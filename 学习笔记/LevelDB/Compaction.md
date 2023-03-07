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

总体可以分为两个部分，如果有immmemtable就先进行minor compaction，然后就是

### 2.1 minor compaction

第一部分minor compaction，将immmemtable落盘，又可以分为三部分，分别为WriteLevel0Table -> LogAndApply -> RemoveObsoleteFiles，依次详细说明：

#### 2.1.1 WriteLevel0Table

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
  Status s = WriteLevel0Table(imm_, &edit, base);		// 写L0文件，并找到这个L0文件应该放在哪个Level
  base->Unref();

  if (s.ok() && shutting_down_.load(std::memory_order_acquire)) {
    s = Status::IOError("Deleting DB during memtable compaction");
  }
  ...
}
```

接下来大致看一下WriteLevel0Table，具体放在读写文件部分详细看。

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

  // 找到这个SSTable应该放在哪个Level
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != nullptr) {
      // 要放的level
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    // 在versionEdit里面添加此文件为新文件
    edit->AddFile(level, meta.number, meta.file_size, meta.smallest,
                  meta.largest);
  }

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);  // 在DBImpl::stats_中添加此stats
  return s;
}
```

之前看学长的面经里有这么一个问题，immmemtable一定落在L0层吗？答案是否，具体原因就要分析一下源码了，这部分的操作就在这个`PickLevelForMemTableOutput()`，代码如下：

```C++
int Version::PickLevelForMemTableOutput(const Slice& smallest_user_key,
                                        const Slice& largest_user_key) {
  int level = 0;
  // 尝试在L0里找，是否有重叠的SSTable，如有，则返回true
  if (!OverlapInLevel(0, &smallest_user_key, &largest_user_key)) {
    InternalKey start(smallest_user_key, kMaxSequenceNumber, kValueTypeForSeek);
    InternalKey limit(largest_user_key, 0, static_cast<ValueType>(0));
    std::vector<FileMetaData*> overlaps;
    while (level < config::kMaxMemCompactLevel) {
      // 看看此Level+1里有没有重复的，如果有的话还是返回Level，因为major Compaction还是比较方便
      if (OverlapInLevel(level + 1, &smallest_user_key, &largest_user_key)) {
        break;
      }
      // 如果Level+1也没有重复的就要去调查Level+2，如果涉及重叠的文件过大，则也还是放在Level合适
      if (level + 2 < config::kNumLevels) {
        // Check that file does not overlap too many grandparent bytes.
        GetOverlappingInputs(level + 2, &start, &limit, &overlaps);
        const int64_t sum = TotalFileSize(overlaps);
        // 详见下面
        if (sum > MaxGrandParentOverlapBytes(vset_->options_)) {
          break;
        }
      }
      level++;
    }
  }
  return level;
}

size_t max_file_size = 2 * 1024 * 1024;

static size_t TargetFileSize(const Options* options) {
  return options->max_file_size;
}

static int64_t MaxGrandParentOverlapBytes(const Options* options) {
  return 10 * TargetFileSize(options);
}
```

接下来分析一下两个判断重叠的函数`OverlapInLevel()`和`GetOverlappingInputs()`。

代码如下：

```C++
bool Version::OverlapInLevel(int level, const Slice* smallest_user_key,
                             const Slice* largest_user_key) {
  return SomeFileOverlapsRange(vset_->icmp_, (level > 0), files_[level],
                               smallest_user_key, largest_user_key);
}

bool SomeFileOverlapsRange(const InternalKeyComparator& icmp,
                           bool disjoint_sorted_files,
                           const std::vector<FileMetaData*>& files,
                           const Slice* smallest_user_key,
                           const Slice* largest_user_key) {
  const Comparator* ucmp = icmp.user_comparator();
  // disjoint_sorted_files = level>0
  if (!disjoint_sorted_files) {
    // 检查L0层里是否有范围重叠的SSTable
    for (size_t i = 0; i < files.size(); i++) {
      const FileMetaData* f = files[i];
      if (AfterFile(ucmp, smallest_user_key, f) ||
          BeforeFile(ucmp, largest_user_key, f)) {
        // No overlap
      } else {
        return true;  // Overlap
      }
    }
    return false;
  }

  // 对于Level>0的部分用二分查找，定位重叠的SSTable
  uint32_t index = 0;
  if (smallest_user_key != nullptr) {
    InternalKey small_key(*smallest_user_key, kMaxSequenceNumber,
                          kValueTypeForSeek);
    // 在读流程里已经提过了，FindFile找到的是首个Key.smallest < files.key.largest的SSTable
    // 因此不能保证Key.largest < files.key.largest
    index = FindFile(icmp, files, small_key.Encode());
  }
	
  // 没有找到
  if (index >= files.size()) {
    // beginning of range is after all files, so no overlap.
    return false;
  }
	
  // 还需要再判断一下Key.largest < files.key.largest
  return !BeforeFile(ucmp, largest_user_key, files[index]);
}
```

然后是`GetOverlappingInputs()`，它的作用是将重叠的SSTable添加到inputs这个vector里，代码如下：

```C++
void Version::GetOverlappingInputs(int level, const InternalKey* begin,
                                   const InternalKey* end,
                                   std::vector<FileMetaData*>* inputs) {
  assert(level >= 0);
  assert(level < config::kNumLevels);
  inputs->clear();
  // 初始化，想要找的范围
  Slice user_begin, user_end;
  if (begin != nullptr) {
    user_begin = begin->user_key();
  }
  if (end != nullptr) {
    user_end = end->user_key();
  }
  
  const Comparator* user_cmp = vset_->icmp_.user_comparator();
  for (size_t i = 0; i < files_[level].size();) {
    FileMetaData* f = files_[level][i++];
    const Slice file_start = f->smallest.user_key();
    const Slice file_limit = f->largest.user_key();
    if (begin != nullptr && user_cmp->Compare(file_limit, user_begin) < 0) {
      // SSTable的最大值都小于要找的最小值
    } else if (end != nullptr && user_cmp->Compare(file_start, user_end) > 0) {
      // SSTable的最小值都大于要找的最大值
    } else {
      inputs->push_back(f);
      if (level == 0) {
        // L0的文件本来就会存在范围重叠，这里需要与找到的文件范围求并集，重新对新区间查找涉及到的文件
        // 因为涉及合并是以文件为单位的，如果并集比原来大，就可能涉及新的文件，就需要重新查找
        
        if (begin != nullptr && user_cmp->Compare(file_start, user_begin) < 0) {
          user_begin = file_start;
          inputs->clear();
          i = 0;
        } else if (end != nullptr &&
                   user_cmp->Compare(file_limit, user_end) > 0) {
          user_end = file_limit;
          inputs->clear();
          i = 0;
        }
      }
    }
  }
}
```

WriteLevel0Table的流程就结束了，写下来就是应用新增的文件形成新版本：

#### 2.1.2 LogAndApply()

```C++
void DBImpl::CompactMemTable() {
  ...
  if (s.ok()) {
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);  // Earlier logs no longer needed
    s = versions_->LogAndApply(&edit, &mutex_);
  }
  ...
}

// 设定一些参数
void SetPrevLogNumber(uint64_t num) {
  has_prev_log_number_ = true;
  prev_log_number_ = num;
}

void SetLogNumber(uint64_t num) {
  has_log_number_ = true;
  log_number_ = num;
}
```

比较复杂的是LogAndApply，下面是代码：

```C++
Status VersionSet::LogAndApply(VersionEdit* edit, port::Mutex* mu) {
  // 查看log_num是否都被设置了
  if (edit->has_log_number_) {
    assert(edit->log_number_ >= log_number_);
    assert(edit->log_number_ < next_file_number_);
  } else {
    edit->SetLogNumber(log_number_);
  }

  if (!edit->has_prev_log_number_) {
    edit->SetPrevLogNumber(prev_log_number_);
  }

  // 这里是为了MVCC吗？不太懂
  edit->SetNextFile(next_file_number_);
  edit->SetLastSequence(last_sequence_);

  // 这里应该算比较核心的部分
  // 新建一个版本，将edit的内容应用于builder，再把builder保存到新版本
  Version* v = new Version(this);
  {
    Builder builder(this, current_);
    builder.Apply(edit);
    builder.SaveTo(v);
  }
  // 这也是核心函数之一，计算各个level分数，作为Compaction的依据
  Finalize(v);

  // Initialize new descriptor log file if necessary by creating
  // a temporary file that contains a snapshot of the current version.
  // 如果没有descriptor_log_就新建manifest文件
  std::string new_manifest_file;
  Status s;
  if (descriptor_log_ == nullptr) {
    // No reason to unlock *mu here since we only hit this path in the
    // first call to LogAndApply (when opening the database).
    assert(descriptor_file_ == nullptr);
    new_manifest_file = DescriptorFileName(dbname_, manifest_file_number_);
    s = env_->NewWritableFile(new_manifest_file, &descriptor_file_);
    if (s.ok()) {
      descriptor_log_ = new log::Writer(descriptor_file_);
      s = WriteSnapshot(descriptor_log_);
    }
  }

  // Unlock during expensive MANIFEST log write
  // 在manifest文件里应用本次更新
  {
    mu->Unlock();

    // Write new record to MANIFEST log
    if (s.ok()) {
      std::string record;
      edit->EncodeTo(&record);
      s = descriptor_log_->AddRecord(record);
      if (s.ok()) {
        s = descriptor_file_->Sync();
      }
      if (!s.ok()) {
        Log(options_->info_log, "MANIFEST write: %s\n", s.ToString().c_str());
      }
    }

    // If we just created a new descriptor file, install it by writing a
    // new CURRENT file that points to it.
    if (s.ok() && !new_manifest_file.empty()) {
      s = SetCurrentFile(env_, dbname_, manifest_file_number_);
    }

    mu->Lock();
  }

  // Install the new version
  if (s.ok()) {
    // 将CURRENT切换到新版本上
    AppendVersion(v);
    log_number_ = edit->log_number_;
    prev_log_number_ = edit->prev_log_number_;
  } else {
    delete v;
    if (!new_manifest_file.empty()) {
      delete descriptor_log_;
      delete descriptor_file_;
      descriptor_log_ = nullptr;
      descriptor_file_ = nullptr;
      env_->RemoveFile(new_manifest_file);
    }
  }

  return s;
}
```

接下来逐一介绍细节：

1. **VersionSet::Builder::Apply()**

   ```C++
   void Apply(const VersionEdit* edit) {
       // Update compaction pointers
       for (size_t i = 0; i < edit->compact_pointers_.size(); i++) {
         const int level = edit->compact_pointers_[i].first;
         vset_->compact_pointer_[level] =
             edit->compact_pointers_[i].second.Encode().ToString();
       }
   
       // Delete files
       for (const auto& deleted_file_set_kvp : edit->deleted_files_) {
         const int level = deleted_file_set_kvp.first;
         const uint64_t number = deleted_file_set_kvp.second;
         levels_[level].deleted_files.insert(number);
       }
   
       // Add new files
       for (size_t i = 0; i < edit->new_files_.size(); i++) {
         const int level = edit->new_files_[i].first;
         FileMetaData* f = new FileMetaData(edit->new_files_[i].second);
         f->refs = 1;
   
         // We arrange to automatically compact this file after
         // a certain number of seeks.  Let's assume:
         //   (1) One seek costs 10ms
         //   (2) Writing or reading 1MB costs 10ms (100MB/s)
         //   (3) A compaction of 1MB does 25MB of IO:
         //         1MB read from this level
         //         10-12MB read from next level (boundaries may be misaligned)
         //         10-12MB written to next level
         // This implies that 25 seeks cost the same as the compaction
         // of 1MB of data.  I.e., one seek costs approximately the
         // same as the compaction of 40KB of data.  We are a little
         // conservative and allow approximately one seek for every 16KB
         // of data before triggering a compaction.
         f->allowed_seeks = static_cast<int>((f->file_size / 16384U));
         if (f->allowed_seeks < 100) f->allowed_seeks = 100;
   
         levels_[level].deleted_files.erase(f->number);
         levels_[level].added_files->insert(f);
       }
     }
   ```

   这里原本的注释已经说的很清楚了，不过要提一嘴的这个allowed_seeks参数设置，机制很有意思，大致翻译一下上面的长注释：

   这里假设一次查找耗时10ms，但是一次读写1MB块也是10ms，一次Compaction会造成25MB的IO，这25MB分别来自：

   1. 读这1MB块
   2. 读10-12MB下一个level的SSTable
   3. 合并后写10-12MB到下一个level的SSTable

   可以得到一个结论：25次查找的开销与1次查找的开销一致，也就是1次seek与40KB的Compaction开销相当，那么也就是说被seek的文件大小每满40KB就可以为其贡献一次seek miss的机会，levelDB为了减少Compaction开销，保守的把40KB调整到16KB，依此计算允许seek miss的次数。

2. **VersionSet::Builder::SaveTo()**

   ```C++
   void SaveTo(Version* v) {
       BySmallestKey cmp;
       cmp.internal_comparator = &vset_->icmp_;
     	// 下面一长串实际上就是对每个level新增SSTable后进行有序排列
       for (int level = 0; level < config::kNumLevels; level++) {
         const std::vector<FileMetaData*>& base_files = base_->files_[level];
         std::vector<FileMetaData*>::const_iterator base_iter = base_files.begin();
         std::vector<FileMetaData*>::const_iterator base_end = base_files.end();
         // FileSet是std
         const FileSet* added_files = levels_[level].added_files;
         v->files_[level].reserve(base_files.size() + added_files->size());
         for (const auto& added_file : *added_files) {
           // Add all smaller files listed in base_
           for (std::vector<FileMetaData*>::const_iterator bpos =
                    std::upper_bound(base_iter, base_end, added_file, cmp);
                base_iter != bpos; ++base_iter) {
             MaybeAddFile(v, level, *base_iter);
           }
   
           MaybeAddFile(v, level, added_file);
         }
   
         // Add remaining base files
         for (; base_iter != base_end; ++base_iter) {
           MaybeAddFile(v, level, *base_iter);
         }
   
   #ifndef NDEBUG
         // Make sure there is no overlap in levels > 0
         if (level > 0) {
           for (uint32_t i = 1; i < v->files_[level].size(); i++) {
             const InternalKey& prev_end = v->files_[level][i - 1]->largest;
             const InternalKey& this_begin = v->files_[level][i]->smallest;
             if (vset_->icmp_.Compare(prev_end, this_begin) >= 0) {
               std::fprintf(stderr, "overlapping ranges in same level %s vs. %s\n",
                            prev_end.DebugString().c_str(),
                            this_begin.DebugString().c_str());
               std::abort();
             }
           }
         }
   #endif
       }
     }
   ```

   

3. **VersionSet::Finalize()**
