```cpp
Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
  // Crete a writer for this update
  Writer w(&mutex_);
  w.batch = updates;
  w.sync = options.sync;
  w.done = false;

  // Create and acquire a mutexlock before pushing current writer to the back of the deque writers_.
  // Only the first writer in deque shall go on the subsequent processing.
  // Other writers will wait here until themselves become the first writer.
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
      // Write to log
      status = log_->AddRecord(WriteBatchInternal::Contents(write_batch));
      bool sync_error = false;
      if (status.ok() && options.sync) {
        status = logfile_->Sync();
        if (!status.ok()) {
          sync_error = true;
        }
      }
      if (status.ok()) {
        // Write to memtable.
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


#Data format

##WriteBatch


|field|sequence_number | count | key_type | key_length |      key      | value_length |       value      |
|-----|----------------|-------|----------|------------|---------------|--------------|------------------|
|bytes|         8      |    4  |     1    |  var_int   |  key_length   |   var_int    | value_length     |
