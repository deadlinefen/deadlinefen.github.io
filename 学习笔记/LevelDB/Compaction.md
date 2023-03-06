# Compaction源码分析！

众所周知，Compaction分为minor Compaction和major Compaction，其中minor Compaction是将immemtable直接编码落盘，major Compaction是对Level L和Level L+1的文件做多路归并。

## 1. 入口函数MaybeScheduleCompaction()

Compaction的入口函数是`MaybeScheduleCompaction()`，看一下它的函数调用：

```C++
void DBImpl::MaybeScheduleCompaction() {
  mutex_.AssertHeld();
  if (background_compaction_scheduled_) {
    // Already scheduled
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // DB is being deleted; no more background compactions
  } else if (!bg_error_.ok()) {
    // Already got an error; no more changes
  } else if (imm_ == nullptr && manual_compaction_ == nullptr &&
             !versions_->NeedsCompaction()) {
    // No work to be done
  } else {
    background_compaction_scheduled_ = true;
    env_->Schedule(&DBImpl::BGWork, this);		// 添加一个后台任务
  }
}

bool NeedsCompaction() const {
  Version* v = current_;
  return (v->compaction_score_ >= 1) || (v->file_to_compact_ != nullptr);
}

void DBImpl::BGWork(void* db) {
  reinterpret_cast<DBImpl*>(db)->BackgroundCall();
}

void DBImpl::BackgroundCall() {
  MutexLock l(&mutex_);
  assert(background_compaction_scheduled_);
  if (shutting_down_.load(std::memory_order_acquire)) {
    // No more background work when shutting down.
  } else if (!bg_error_.ok()) {
    // No more background work after a background error.
  } else {
    BackgroundCompaction();  // 后台Compaction入口
  }

  background_compaction_scheduled_ = false;

  // Previous compaction may have produced too many files in a level,
  // so reschedule another compaction if needed.
  MaybeScheduleCompaction();	// 再次查询是否需要compaction
  background_work_finished_signal_.SignalAll();
}
```

其他的调用点：

1. 读完之后，如果有L0 SSTable多次不命中就及时将它Compaction

   ```C++
   Status DBImpl::Get(const ReadOptions& options, const Slice& key,
                      std::string* value) {
   	...
   	if (have_stat_update && current->UpdateStats(stats)) {
       MaybeScheduleCompaction();
     }
     ...
   }
   ```

2. 写的时候的MakeRoomForWrite

   ```C++
   // Soft limit on number of level-0 files.  We slow down writes at this point.
   static const int kL0_SlowdownWritesTrigger = 8;
   
   // Maximum number of level-0 files.  We stop writes at this point.
   static const int kL0_StopWritesTrigger = 12;
   
   Status DBImpl::MakeRoomForWrite(bool force) {
     mutex_.AssertHeld();
     assert(!writers_.empty());
     bool allow_delay = !force;
     Status s;
     while (true) {
       if (!bg_error_.ok()) {
         // Yield previous error
         s = bg_error_;
         break;
       } else if (allow_delay && versions_->NumLevelFiles(0) >=
                                     config::kL0_SlowdownWritesTrigger) {
         // 当L0文件数超过8时，需要放慢写入
         mutex_.Unlock();
         env_->SleepForMicroseconds(1000);
         allow_delay = false;  // Do not delay a single write more than once
         mutex_.Lock();
       } else if (!force &&
                  (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {
         // 空间还够用，即memtable大小小于4M，这时直接往memtable里写
         break;
       } else if (imm_ != nullptr) {
         // immmemtable存在，此时memtable的size已经超限，需要等待immmemtable落盘
         Log(options_.info_log, "Current memtable full; waiting...\n");
         background_work_finished_signal_.Wait();
       } else if (versions_->NumLevelFiles(0) >= config::kL0_StopWritesTrigger) {
         // 太多L0文件，需要等待compaction结束
         Log(options_.info_log, "Too many L0 files; waiting...\n");
         background_work_finished_signal_.Wait();
       } else {
         // 将memtable固定为immmemtable，新建memtable
         assert(versions_->PrevLogNumber() == 0);
         uint64_t new_log_number = versions_->NewFileNumber();
         WritableFile* lfile = nullptr;
         s = env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile);
         if (!s.ok()) {
           // Avoid chewing through file number space in a tight loop.
           versions_->ReuseFileNumber(new_log_number);
           break;
         }
         delete log_;
         delete logfile_;
         logfile_ = lfile;
         logfile_number_ = new_log_number;
         log_ = new log::Writer(lfile);
         imm_ = mem_;
         has_imm_.store(true, std::memory_order_release);
         mem_ = new MemTable(internal_comparator_);
         mem_->Ref();
         force = false;  // Do not force another compaction if have room
         MaybeScheduleCompaction();		// 触发compaction
       }
     }
     return s;
   }
   ```

3. 打开数据库时，看看需不需要compaction

   ```C++
   Status DB::Open(const Options& options, const std::string& dbname, DB** dbptr) {
     ...
     if (s.ok()) {
       impl->RemoveObsoleteFiles();
       impl->MaybeScheduleCompaction();
     }
     ...
   }
   ```

## 2. 核心函数BackgroundCompaction()

第一部分minor compaction，将immmemtable落盘

```C++
void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

  if (imm_ != nullptr) {
    CompactMemTable();
    return;
  }
	...
}

void DBImpl::CompactMemTable() {
  mutex_.AssertHeld();
  assert(imm_ != nullptr);

  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();
  Status s = WriteLevel0Table(imm_, &edit, base);		// 写L0文件
  base->Unref();

  if (s.ok() && shutting_down_.load(std::memory_order_acquire)) {
    s = Status::IOError("Deleting DB during memtable compaction");
  }
  
  if (s.ok()) {
    edit.SetPrevLogNumber(0);				// 上一个log num？
    edit.SetLogNumber(logfile_number_);  // redo log的file number
    s = versions_->LogAndApply(&edit, &mutex_);  // 应用这个版本
  }

  if (s.ok()) {
    imm_->Unref();
    imm_ = nullptr;
    has_imm_.store(false, std::memory_order_release);
    RemoveObsoleteFiles();		// 清理垃圾文件
  } else {
    RecordBackgroundError(s);
  }
}
```

很好，非常符合我对minor compaction的想象，接下来大致看一下WriteLevel0Table，具体放在读写文件部分详细看。

```C++
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  mutex_.AssertHeld();
  const uint64_t start_micros = env_->NowMicros();
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();		// 获得一个新的文件号
  pending_outputs_.insert(meta.number);				// 插入工作set
  Iterator* iter = mem->NewIterator();				// 生成memtable的迭代器
  Log(options_.info_log, "Level-0 table #%llu: started",
      (unsigned long long)meta.number);

  Status s;
  {
    mutex_.Unlock();
    // build table，详见文件读写
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }

  Log(options_.info_log, "Level-0 table #%llu: %lld bytes %s",
      (unsigned long long)meta.number, (unsigned long long)meta.file_size,
      s.ToString().c_str());
  delete iter;
  pending_outputs_.erase(meta.number);  // 移除工作的set

  // Note that if file_size is zero, the file has been deleted and
  // should not be added to the manifest.
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != nullptr) {
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    edit->AddFile(level, meta.number, meta.file_size, meta.smallest,
                  meta.largest);
  }

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);
  return s;
}
```



