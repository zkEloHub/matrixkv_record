### nvm_mod

**bitmap.cc**

记录 L0 中 Table 使用情况



**column_compaction_iterator.h**

对一个 `FileEntry` 中的 KV 数据进行访问 （迭代器）

私有数据:

```cpp
const InternalKeyComparator* icmp_;
char *raw_data_;            // nvm 地址
FileEntry* file_;           // 对应的 FileEntry
vector<Slice> vKey_;
vector<Slice> vValue_;
int current_;               // vkey_ 中当前正在处理的 key 下标
```

FileEntry 结构:

```cpp
struct FileEntry {
    uint64_t filenum;
    int sstable_index; // 当前 File 在 nvm 中的 index; (nvm 中根据 index 可计算 row 起始地址)
    struct KeysMetadata* keys_meta = nullptr; //指向多个（keys_num个）连续内存的KeysMetadata;  keys_meta 中的数据 按照 key 有序
    uint64_t keys_num;			// Key 数量
    uint64_t key_point_filenum; //key 指向下一个文件的filenum (防止中间删除了文件)
};
```



**column_compaction.h**

一个 `Column Compaction` 所对应的一个实例

私有数据:

```cpp
struct ColumnCompactionItem {
    std::vector<FileEntry*> files;  // 本次 compaction 的 FileEntry 信息
    std::vector<uint64_t> keys_num; // 每个 FileEntry 对应要 compaction 的总 key num
    std::vector<uint64_t> keys_size;// 每个 FileEntry 对应要 compaction 的总 key size 
    uint64_t L0select_size;         // level0 中本次 compaction 的总数据量

    std::vector<FileMetaData*> L0compactionfiles;// 本次 compaction 中 L0 的 FileMetaData 信息
    std::vector<FileMetaData*> L1compactionfiles;// 本次 compaction 中 L1 的 FileMetaData 信息 (overlap)
	// Level-0 Key Range
    InternalKey L0smallest;
    InternalKey L0largest;
};
```



**keys_merge_iterator.h**

对多个 FileEntry 的 KV 数据进行访问 (迭代器)



### NVM Flush

**DBImpl::MaybeScheduleFlushOrCompaction**

- 根据是否有 column family 需要 flush, 以及是否满足 flush 条件，来调度 DBImpl::BGWorkFlush

- 根据是否有 column family 需要 compaction, 以及是否满足 compaction 条件，来调度 DBImpl::BGWorkCompaction
  - 条件：存在未调度任务 (flush_queu\_, compaction_queue\_);  后台任务量未超出限



**DBImpl::BGWorkFlush => DBImpl::BackgroundCallFlush**

- 分配 JOB 任务
- 调用 DBImpl::BackgroundFlush



**DBImpl::BackgroundFlush**

- 处理 flush_queue\_ 中需要 flush 的 Column Family
  - L0 中的 Column Family : DBImpl::FlushMemTablesToNvm
  - Ln (n > 0) 中的 Column Family : DBImpl::FlushMemTablesToOutputFiles
- 清除已丢弃的 Column Family



**DBImpl::FlushMemTablesToNvm => DBImpl::FlushMemTableToNvm**

- 生成一个 NvmFlushJob 对象 flush_job (用于一次 Flush 任务)
- flush_job.**PickMemTable**: 选出本次 Flush 任务中需要 Flush 的 MemTable
  - MemTableList::PickMemtablesToFlush: 
    - 从 current\_->memlist\_ 中取出一个需要 flush 的 immutable
    - nvmFlush 每次 Flush 任务只处理一个 immutable, 退出返回
  - 更新 flush_job.**mems_**

- 调用 flush_job.Run，执行 Flush 任务
  - NvmFlushJob::**WriteLevel0Table**

- 记录 flush 之后的信息到 file_meta



**NvmFlushJob::WriteLevel0Table**

- 将 mems_ 处理成 internal iterators 的形式: **memtables**
- 将 memtables 转化成 ScopedArenaIterator， 调用 **BuildInsertNvm** 将 KV 数据持久化



**NvmFlushJob::BuildInsertNvm**

- nvm 中分配 Table 空间
- 构建 L0builder， 调用 Add 方法，将 KV 持久化到 nvm (buffer)
- 完成所有 kv 的 Add 操作，L0builder->Finish,  实现真正持久化
- 将持久化后的 Table 信息更新到 flush_job.**meta\_**





















