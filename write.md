# Data format

## WriteBatch


|field|sequence_number | count | key_type |   key_length  |      key      |  value_length   |       value      |
|-----|----------------|-------|----------|---------------|---------------|-----------------|------------------|
|bytes|         8      |    4  |     1    |  var_int_32   |  key_length   |   var_int_32    |   value_length   |


## log::Writer
| field |  checksum  | WriteBatch_data_length |  record_type(full/first/middle/last)  |      WriteBatch_data     |
|-------|------------|------------------------|---------------------------------------|--------------------------|
| bytes |      4     |            2           |                    1                  |  WriteBatch_data_length  |


## MemTable

| field | key_length |     key     |   tag (sequence_number << 8 \| key_type)   |   value_length   |     value   |
|-------|------------|-------------|--------------------------------------------|------------------|-------------|
| bytes | var_int_32 | key_length  |                      8                     |     var_int_32   | value_length|


# Code Flow

## Entry

```cpp
// db_impl.cc
Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
  // Crete a writer for this update
  Writer w(&mutex_);
  w.batch = updates;
  w.sync = options.sync;
  w.done = false;

  // Create and acquire a mutexlock before pushing current writer to the back of the deque writers_.
  // Only the first writer in deque shall go on the subsequent processing.
  // Others will wait here until the first writer sends signal to them.
  // This behavior is controlled by condition variable in standard library.
  MutexLock l(&mutex_);
  writers_.push_back(&w);
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait();
  }
  if (w.done) {
    return w.status;
  }

  // May temporarily unlock and wait.
  Status status = MakeRoomForWrite(updates == nullptr);
  uint64_t last_sequence = versions_->LastSequence();
  Writer* last_writer = &w;
  if (status.ok() && updates != nullptr) {  // nullptr batch is for compactions
    // BuildBatchGroup() helps merge as many data of writers on deque as possible from front() to end().
    // Then, write_batch contains all the merged data which shall be written to log and memtable.
    // last_writer points to the last writer whose data is merged.
    WriteBatch* write_batch = BuildBatchGroup(&last_writer);
    WriteBatchInternal::SetSequence(write_batch, last_sequence + 1);
    last_sequence += WriteBatchInternal::Count(write_batch);

    // Add to log and apply to memtable.  We can release the lock
    // during this phase since &w is currently responsible for logging
    // and protects against concurrent loggers and concurrent writes
    // into mem_.
    {
      // When leveldb write merged data to log and memtable, it's allowed to
      // add more writer to the end of deque. Thus, we can do mutex unlock here.
      mutex_.Unlock();
      // Firstly, write to log
      status = log_->AddRecord(WriteBatchInternal::Contents(write_batch));
      bool sync_error = false;
      if (status.ok() && options.sync) {
        status = logfile_->Sync();
        if (!status.ok()) {
          sync_error = true;
        }
      }
      if (status.ok()) {
        // Secondly, write to memtable.
        status = WriteBatchInternal::InsertInto(write_batch, mem_);
      }
      mutex_.Lock();
      if (sync_error) {
        // The state of the log file is indeterminate: the log record we
        // just added may or may not show up when the DB is re-opened.
        // So we force the DB into a mode where all future writes fail.
        RecordBackgroundError(status);
      }
    }
    if (write_batch == tmp_batch_) tmp_batch_->Clear();

    versions_->SetLastSequence(last_sequence);
  }

  // Pop all the writers whose data was written to log and memtable.
  // If current writer is not the first one, then send condition variable signal
  // to inform it - Your work is done. End yourself.
  while (true) {
    Writer* ready = writers_.front();
    writers_.pop_front();
    if (ready != &w) {
      ready->status = status;
      ready->done = true;
      ready->cv.Signal();
    }
    if (ready == last_writer) break;
  }

  // Notify new head of write queue
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }

  return status;
}
```

## Write Log

```cpp
// log_writer.cc
// Slice is divided by blocks. Each block is 32KB.
// If the head and tail of the slice is in this block, then mark the type as kFullType.
// If neither head nor tail is in this block, then kMiddleType.
// ..etc.
Status Writer::AddRecord(const Slice& slice) {
  size_t left = slice.size();

  // Fragment the record if necessary and emit it.  Note that if slice
  // is empty, we still want to iterate once to emit a single
  // zero-length record
  do {
    if (leftover < kHeaderSize) {
      // Switch to a new block
      if (leftover > 0) {
        // Fill the trailer (literal below relies on kHeaderSize being 7)
        dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover));
      }
      block_offset_ = 0;
    }

    if (begin && end) {
      type = kFullType;
    } else if (begin) {
      type = kFirstType;
    } else if (end) {
      type = kLastType;
    } else {
      type = kMiddleType;
    }

    s = EmitPhysicalRecord(type, ptr, fragment_length);
  } while (s.ok() && left > 0);
  return s;
}

Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr,
                                  size_t length) {
  // Format the header
  char buf[kHeaderSize];
  buf[4] = static_cast<char>(length & 0xff);
  buf[5] = static_cast<char>(length >> 8);
  // ...

  // Write the header
  Status s = dest_->Append(Slice(buf, kHeaderSize));
  if (s.ok()) {
    // Write the payload
    s = dest_->Append(Slice(ptr, length));
    if (s.ok()) {
      s = dest_->Flush();
    }
  }
  
  // ...
}
```

```cpp
// env_posix.cc
Status Append(const Slice& data) override {
  // ...
  
  // If buffer is big enough for storing data, then done.
  if (write_size == 0) {
    return Status::OK();
  }

  // Can't fit in buffer, so need to do at least one write.
  Status status = FlushBuffer();
  if (!status.ok()) {
    return status;
  }

  // Small writes go to buffer for later write to log file
  // (by the call of Flush()/FlushBuffer() or WriteUnbuffered()).
  if (write_size < kWritableFileBufferSize) {
    std::memcpy(buf_, write_data, write_size);
    pos_ = write_size;
    return Status::OK();
  }
  // Large writes are written to the log file directly.
  return WriteUnbuffered(write_data, write_size);
}

Status WriteUnbuffered(const char* data, size_t size) {
  while (size > 0) {
    // ::write is defined in unistd.h of c posix library.
    // https://en.wikipedia.org/wiki/C_POSIX_library
    ssize_t write_result = ::write(fd_, data, size);
    // ...
  }
  return Status::OK();
}
```
