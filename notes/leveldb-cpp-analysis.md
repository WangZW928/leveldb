# Deep Analysis of LevelDB: Advanced C++ Techniques and Engineering Philosophy

> 📐 [查看架构图](leveldb-architecture.html)

## 1. Overview and Build

LevelDB is a persistent, ordered key-value store implemented as a log-structured merge tree. Its public surface is deliberately small: `DB`, `Options`, `ReadOptions`, `WriteOptions`, `WriteBatch`, `Iterator`, `Slice`, `Status`, `Comparator`, `FilterPolicy`, `Cache`, and `Env`. Most implementation detail lives outside `include/`, primarily in `db/`, `table/`, `util/`, `port/`, and `helpers/`.

At the highest level:

- writes append to a write-ahead log and insert into an in-memory skip-list memtable;
- full memtables become immutable and are flushed to sorted string table files;
- reads merge current memtable, immutable memtable, and versioned SSTable files;
- compaction rewrites overlapping sorted files into lower levels while discarding hidden versions and obsolete deletions.

### Build Verification

Requested command:

```sh
mkdir -p build && cd build && cmake .. && make -j$(nproc)
```

Result in this checkout: CMake failed because both vendored submodule directories are empty placeholders:

```text
third_party/googletest does not contain a CMakeLists.txt file
third_party/benchmark does not contain a CMakeLists.txt file
```

`git submodule status` shows both submodules uninitialized:

```text
-f7547e29ccaed7b64ef4f7495ecfff1c9f6f3d03 third_party/benchmark
-662fe38e44900c007eccb65a5d2ea19df7bd520e third_party/googletest
```

Because network access is unavailable in this environment, I could not populate those submodules. I verified the core library and utility binary offline by disabling test and benchmark targets and selecting C++17:

```sh
rm -rf build
mkdir -p build
cd build
cmake -DCMAKE_CXX_STANDARD=17 \
      -DLEVELDB_BUILD_TESTS=OFF \
      -DLEVELDB_BUILD_BENCHMARKS=OFF ..
make -j$(nproc)
```

This built successfully:

```text
[ 95%] Built target leveldb
[100%] Built target leveldbutil
```

The C++17 flag is necessary in this checkout because `util/no_destructor.h` and the platform singleton code use `std::is_standard_layout_v`, which is a C++17 variable template. Without `-DCMAKE_CXX_STANDARD=17`, the build fails under GCC 13.3.0.

Running:

```sh
cd build && ctest --output-on-failure
```

reported:

```text
No tests were found!!!
```

That is expected because test target generation had to be disabled to work around the missing `googletest` submodule. To run the full test suite in a complete checkout, initialize submodules first, then configure with tests enabled:

```sh
git submodule update --init --recursive
rm -rf build
mkdir build && cd build
cmake -DCMAKE_CXX_STANDARD=17 ..
make -j$(nproc)
ctest --output-on-failure
```

### Repository Map

- `include/leveldb/`: stable public C++ API and ABI-facing declarations.
- `db/`: database engine, write path, recovery, WAL, memtable, snapshots, version metadata, compaction, repair, C API.
- `table/`: SSTable reader/writer, block encoding, block iterators, filter blocks, two-level iterators.
- `util/`: arena allocator, cache, Bloom filter, coding primitives, CRC32C, env helpers, logging, status.
- `port/`: platform abstraction for mutexes, condition variables, compression, CRC acceleration, thread annotations.
- `helpers/`: optional helper implementations such as `memenv`.
- `doc/`: design notes and persistent format documentation.
- `benchmarks/`, `issues/`, `*_test.cc`: tests and benchmarks, dependent on missing third-party test libraries in this checkout.

## 2. Architecture Map

```text
Public API
include/leveldb/db.h
include/leveldb/options.h
include/leveldb/iterator.h
include/leveldb/env.h
include/leveldb/status.h
include/leveldb/slice.h
        |
        v
DBImpl (db/db_impl.{h,cc})
  - owns current mutable MemTable
  - owns optional immutable MemTable
  - owns WAL writer
  - owns VersionSet
  - schedules background compaction through Env
        |
        +---------------------+
        |                     |
        v                     v
MemTable                 VersionSet / Version
db/memtable.*            db/version_set.*
  - SkipList               - file metadata
  - Arena                  - MANIFEST edits
  - internal keys          - compaction picking
        |                     |
        v                     v
BuildTable              TableCache
db/builder.cc           db/table_cache.cc
        |                     |
        v                     v
TableBuilder            Table
table/table_builder.cc  table/table.cc
  - data blocks             - footer/index
  - index block             - block cache
  - filter block            - Bloom filter checks
  - footer                  - two-level iteration
```

### LSM Tree Manifestation in Code

LevelDB's LSM structure appears through these concrete objects:

- mutable memtable: `DBImpl::mem_`, a `MemTable` backed by `SkipList<const char*, KeyComparator>`;
- immutable memtable: `DBImpl::imm_`, the previous full memtable waiting for flush;
- write-ahead log: `DBImpl::log_` and `DBImpl::logfile_`, written before memtable mutation;
- levels: `Version::files_[config::kNumLevels]`, with seven levels by default;
- level policy: constants in `db/dbformat.h`, such as `kL0_CompactionTrigger`, `kL0_SlowdownWritesTrigger`, `kL0_StopWritesTrigger`, and `kMaxMemCompactLevel`;
- manifest/versioning: `VersionEdit` records added/deleted files and `VersionSet::LogAndApply()` atomically publishes metadata;
- compaction: `VersionSet::PickCompaction()`, `DBImpl::BackgroundCompaction()`, and `DBImpl::DoCompactionWork()`.

Level 0 is special: files may overlap because they are direct flushes of memtables. Levels greater than zero maintain non-overlapping key ranges, which allows binary search over file metadata and cheaper lookup.

## 3. Write Path and Read Path

### Write Path

```text
DB::Put / DB::Delete
        |
        v
temporary WriteBatch
        |
        v
DBImpl::Write
  - enqueue Writer
  - wait until at front
  - MakeRoomForWrite
  - BuildBatchGroup
  - assign sequence numbers
        |
        +--> log_->AddRecord(batch bytes)
        |       |
        |       +--> optional logfile_->Sync()
        |
        +--> WriteBatchInternal::InsertInto
                |
                v
             MemTable::Add
                |
                v
             SkipList insert in Arena memory
```

Important details:

- `DB::Put()` and `DB::Delete()` are convenience methods implemented in `db/db_impl.cc`; each creates a `WriteBatch` and calls virtual `Write()`.
- `DBImpl::Write()` serializes writers through `writers_`, a `std::deque<Writer*>` guarded by `mutex_`.
- Adjacent writers may be combined by `BuildBatchGroup()` up to about 1 MiB, with a smaller limit for small leading writes to avoid unfair latency.
- `WriteBatchInternal::SetSequence()` assigns the first sequence number in the batch; each record gets a monotonically increasing sequence.
- The WAL is appended before the memtable is changed. With `WriteOptions::sync`, the writable file is explicitly synced before success.
- `WriteBatchInternal::InsertInto()` replays the batch through a `WriteBatch::Handler`, producing `MemTable::Add(seq, type, key, value)` calls.
- `MemTable::Add()` stores:

```text
varint32 internal_key_size
user_key bytes
fixed64 sequence/type tag
varint32 value_size
value bytes
```

When the memtable exceeds `options_.write_buffer_size`, `MakeRoomForWrite()` rotates:

```text
old mem_ -> imm_
new log file
new empty mem_
MaybeScheduleCompaction()
```

The immutable memtable is later flushed by `DBImpl::CompactMemTable()` through `WriteLevel0Table()` and `BuildTable()`.

### Read Path

```text
DBImpl::Get
        |
        v
choose snapshot sequence
        |
        v
LookupKey(user_key, snapshot)
        |
        +--> mem_->Get
        |
        +--> imm_->Get
        |
        +--> Version::Get
                |
                v
          ForEachOverlapping file
                |
                v
          TableCache::Get
                |
                v
          Table::InternalGet
                |
                +--> index block seek
                +--> filter block KeyMayMatch
                +--> data block read/cache
                +--> block iterator seek
```

Important details:

- `DBImpl::Get()` locks only long enough to select and ref-count `mem_`, `imm_`, and the current `Version`; it unlocks while searching.
- The lookup key contains the requested user key plus the selected sequence number and `kValueTypeForSeek`.
- Internal key ordering is by increasing user key and decreasing sequence number, so the first matching internal key is the newest visible version.
- If neither memtable has the key, `Version::Get()` searches overlapping SSTables.
- For Level 0, multiple overlapping files may be checked. For levels greater than zero, non-overlap lets the code binary-search to at most one candidate file per level.
- `Table::InternalGet()` first seeks the index block, then consults the filter block. If the Bloom filter says "definitely absent", the data block is not read.
- `Table::BlockReader()` integrates the block cache. Cache keys combine a per-table cache id and block offset.
- Reads can trigger compaction pressure: if a lookup has to seek through more than one file, `Version::UpdateStats()` may decrement a file's `allowed_seeks` and schedule compaction.

### Iterator Read Path

```text
DBImpl::NewIterator
        |
        v
DBImpl::NewInternalIterator
  - memtable iterator
  - immutable memtable iterator
  - table iterators from current Version
        |
        v
NewMergingIterator over internal keys
        |
        v
NewDBIterator
  - hides overwritten versions
  - hides deletion markers
  - enforces snapshot sequence
```

This is a canonical composition of iterators: no single source must know about all other sources. The database-level iterator is an adapter over a merging internal iterator.

### Compaction

```text
MaybeScheduleCompaction
        |
        v
Env::Schedule(DBImpl::BGWork)
        |
        v
DBImpl::BackgroundCompaction
        |
        +--> if imm_ exists: CompactMemTable()
        |
        +--> else choose compaction:
                - manual: VersionSet::CompactRange()
                - automatic: VersionSet::PickCompaction()
        |
        +--> trivial move if possible
        |
        +--> DoCompactionWork()
                |
                v
          merge input iterators
          drop hidden versions
          drop obsolete tombstones
          write new TableBuilder outputs
          InstallCompactionResults()
```

Manual compaction comes from `DB::CompactRange()` and test hooks such as `TEST_CompactRange()`. Automatic compaction is scheduled when:

- an immutable memtable exists;
- `VersionSet::NeedsCompaction()` is true because a level's score is high;
- read sampling marks a file as seek-expensive;
- manual compaction state is present.

The critical compaction logic is in `DBImpl::DoCompactionWork()`. It maintains:

- `smallest_snapshot`: the oldest live snapshot sequence;
- `current_user_key`: the user key currently being processed;
- `last_sequence_for_key`: newest sequence seen for that user key.

It drops:

- older versions hidden by a newer version visible to every live snapshot;
- deletion markers whose sequence is older than the oldest snapshot and whose user key has no data in higher levels.

This is the heart of LSM garbage collection.

## 4. C++ Advanced Techniques

### RAII Patterns

LevelDB uses RAII selectively and pragmatically rather than dogmatically.

`util/mutexlock.h` is the clearest RAII type:

```cpp
class MutexLock {
 public:
  explicit MutexLock(port::Mutex* mu) : mu_(mu) { mu_->Lock(); }
  ~MutexLock() { mu_->Unlock(); }
 private:
  port::Mutex* const mu_;
};
```

Examples:

- `DBImpl::Get()` uses `MutexLock l(&mutex_)` to guard selection/ref-counting of shared state.
- `DBImpl::GetSnapshot()` and `ReleaseSnapshot()` use the same scoped lock pattern.
- `LRUCache` methods use `MutexLock` around cache state.

Other RAII-like ownership patterns:

- `Status` owns a heap-allocated error state and frees it in `~Status()`.
- `Arena` owns allocated blocks and frees all blocks in `~Arena()`.
- `Table::Rep::~Rep()` deletes filter, filter data, and index block.
- `Iterator::RegisterCleanup()` attaches destruction callbacks to an iterator, so cache handles and ref-counted state are released when the iterator is destroyed.
- `port::CondVar::Wait()` uses `std::unique_lock<std::mutex>` with `std::adopt_lock` to integrate LevelDB's explicit mutex protocol with the standard condition variable.

Notably, LevelDB still uses many raw pointers because it predates broad modern C++ smart-pointer practice and because ownership crosses C-compatible APIs. The contracts are explicit in comments: "caller should delete", "result belongs to leveldb", "caller must release", and so on.

### Iterator Pattern

The public iterator abstraction is `include/leveldb/iterator.h`:

```cpp
class Iterator {
 public:
  virtual bool Valid() const = 0;
  virtual void SeekToFirst() = 0;
  virtual void SeekToLast() = 0;
  virtual void Seek(const Slice& target) = 0;
  virtual void Next() = 0;
  virtual void Prev() = 0;
  virtual Slice key() const = 0;
  virtual Slice value() const = 0;
  virtual Status status() const = 0;
};
```

Concrete implementations include:

- `MemTableIterator` in `db/memtable.cc`, adapting `SkipList::Iterator`;
- `Block::Iter` in `table/block.cc`, decoding prefix-compressed block entries;
- `TwoLevelIterator` in `table/two_level_iterator.cc`, mapping index entries to block iterators;
- `MergingIterator` in `table/merger.cc`, merging sorted child iterators;
- `DBIter` in `db/db_iter.cc`, hiding internal sequence/type details from users.

Two design choices are especially important:

- iterators return `Slice` views into internal storage, avoiding copies in hot paths;
- `RegisterCleanup()` decouples iterator lifetime from cache and version lifetime.

For example, `Table::BlockReader()` registers cleanup that either deletes an uncached block or releases a block-cache handle. `DBImpl::NewInternalIterator()` registers cleanup that unreferences `mem_`, `imm_`, and the current version after the iterator dies.

### Builder Pattern: `TableBuilder` and `BlockBuilder`

`TableBuilder` incrementally constructs a complete SSTable:

```text
Add sorted key/value pairs
  -> accumulate prefix-compressed data block
  -> flush data block at target size
  -> add delayed shortened index key
  -> collect filter keys
Finish
  -> filter block
  -> metaindex block
  -> index block
  -> footer
```

`TableBuilder::Rep` is an internal state bundle containing options, output file, offset, current data block, index block, filter block, last key, status, and pending index-entry state.

The delayed index entry is elegant. `TableBuilder` waits until it has seen the first key of the next block before writing the index key for the previous block. That lets it call `Comparator::FindShortestSeparator()` to produce a shorter separator key that is still greater than all keys in the previous block and less than keys in the next block.

`BlockBuilder` constructs a block using prefix compression:

```text
shared_bytes: varint32
unshared_bytes: varint32
value_length: varint32
key_delta
value
...
restart_offsets: fixed32[]
num_restarts: fixed32
```

Every `block_restart_interval` entries, compression restarts with a full key. This gives a controlled tradeoff: good compression within a block, plus logarithmic seek to a restart point followed by short linear scan.

### Factory and Strategy Pattern

LevelDB uses abstract interfaces as strategy objects:

- `Env`: filesystem, locking, scheduling, sleeping, logging, time;
- `Comparator`: key ordering and index-key shortening;
- `FilterPolicy`: probabilistic membership filter;
- `Cache`: block/table cache behavior;
- `Logger`: logging sink.

`Env::Default()` is a factory for the platform environment. `NewLRUCache()` and `NewBloomFilterPolicy()` are factories for built-in strategies. `Options` wires strategies into the engine:

```cpp
options.env = Env::Default();
options.comparator = BytewiseComparator();
options.block_cache = NewLRUCache(...);
options.filter_policy = NewBloomFilterPolicy(10);
```

The payoff is testability and portability. `helpers/memenv` supplies an in-memory `Env`, tests inject custom environments, and platform code is isolated behind `port/`.

### Arena Allocator

`util/arena.{h,cc}` implements a simple monotonic allocator:

- small allocations bump `alloc_ptr_`;
- fallback allocates a new 4096-byte block;
- large allocations greater than one quarter of a block get their own block;
- `AllocateAligned()` adds slop to satisfy at least pointer-size or 8-byte alignment;
- memory is freed all at once in `Arena::~Arena()`.

This is ideal for memtables and skip-list nodes because:

- nodes are never individually deleted;
- allocation is on a hot write path;
- object lifetime equals memtable lifetime;
- readers can safely traverse nodes because nodes do not disappear.

Comparison with AMReX-style arenas: AMReX arenas are designed for high-performance scientific workloads with heterogeneous memory spaces, device/host pools, managed memory, pinned memory, and reuse across kernels. LevelDB's arena is far narrower: a CPU-only bump allocator for many tiny objects with database-lifetime grouping. AMReX optimizes memory placement and reuse across accelerator-aware execution. LevelDB optimizes allocator overhead and pointer stability in a concurrent in-memory index.

### Skip List

`db/skiplist.h` is a template:

```cpp
template <typename Key, class Comparator>
class SkipList { ... };
```

It is used by `MemTable` as:

```cpp
typedef SkipList<const char*, KeyComparator> Table;
```

Concurrency model:

- writes require external synchronization;
- reads need only a guarantee that the skip list is not destroyed;
- nodes are never deleted until the entire skip list is destroyed;
- inserted node contents are immutable after publication;
- next pointers are `std::atomic<Node*>`;
- publication uses release stores and traversal uses acquire loads.

This is not a fully lock-free multi-writer skip list. It is a single-writer or externally synchronized writer structure with lock-free readers. That matches LevelDB's write architecture: DB writes are serialized through `DBImpl::Write()`, while reads can traverse the memtable concurrently.

The random height distribution uses branching factor 4 and max height 12. Node memory uses a flexible-array style layout:

```cpp
char* p = arena_->AllocateAligned(sizeof(Node) +
    sizeof(std::atomic<Node*>) * (height - 1));
return new (p) Node(key);
```

That is placement new over arena memory, with the trailing `next_` array sized by node height.

### Bloom Filter

`util/bloom.cc` implements `FilterPolicy`. It chooses:

- bits per key from user input;
- number of probes `k ~= bits_per_key * ln(2)`, clamped to `[1, 30]`;
- double hashing to avoid computing `k` independent hashes.

`table/filter_block.cc` groups filters by data-block offset. The default base is 2 KiB of file offset:

```cpp
static const size_t kFilterBaseLg = 11;
```

Integration:

- `TableBuilder::Add()` passes every key to `FilterBlockBuilder::AddKey()`;
- `TableBuilder::Flush()` calls `StartBlock(offset)` so filters align with data ranges;
- `Table::ReadMeta()` finds the metaindex entry named `filter.<policy name>`;
- `Table::InternalGet()` decodes the target data block handle and asks `filter->KeyMayMatch(handle.offset(), k)`;
- if the filter returns false, LevelDB avoids the data block read.

The internal filter wrapper in `db/dbformat.cc` strips sequence/type suffixes so filters operate on user keys rather than internal keys.

### CRC32C

`util/crc32c.{h,cc}` supplies masked CRC32C checksums for blocks and logs.

Important points:

- `crc32c::Extend()` computes CRC of concatenated data;
- `crc32c::Mask()` rotates and adds a constant before storing, avoiding problematic embedded CRC self-interaction;
- `port::AcceleratedCRC32C()` is used when available;
- otherwise a portable table-based implementation processes bytes and 16-byte strides.

The build in this environment reported:

```text
Looking for crc32c_value in crc32c - not found
```

so this build uses the software fallback.

### Coding and Varints

`util/coding.{h,cc}` defines endian-neutral persistent encodings:

- fixed integers are little-endian;
- varints use 7 data bits per byte and the high bit as continuation;
- strings are `varint32 length` followed by bytes.

For example, `EncodeVarint32()` has fast explicit cases for 1 through 5 bytes. `GetVarint32Ptr()` has a fast one-byte path before falling back to the loop. This is systems C++ in miniature: stable wire format, no locale, no stream overhead, explicit bounds checks where needed.

### `Slice`: Pre-`std::string_view`

`include/leveldb/slice.h` is essentially a pre-standard `string_view`:

```cpp
class Slice {
 public:
  Slice(const char* d, size_t n) : data_(d), size_(n) {}
  Slice(const std::string& s) : data_(s.data()), size_(s.size()) {}
  const char* data() const;
  size_t size() const;
  void remove_prefix(size_t n);
  std::string ToString() const;
};
```

It is intentionally copyable and non-owning. The contract is explicit: the referenced storage must outlive the `Slice`. In exchange, almost every hot-path API avoids copying key/value bytes.

### Pimpl Idiom

There is a local pimpl-like idiom in several classes:

- `TableBuilder` owns `TableBuilder::Rep* rep_`;
- `Table` owns `Table::Rep* rep_`;
- C API wrapper types hold C++ implementation pointers.

This hides implementation details from headers and keeps public or semi-public class layouts small. It is not used everywhere. Internal implementation classes such as `DBImpl`, `Version`, and `MemTable` expose their fields in private headers because they are not public ABI.

### Template Use

The most important template is `SkipList<Key, Comparator>`. It is generic enough to accept arbitrary key and comparator types but specialized enough to compile down to direct comparator calls.

Other template use:

- `NoDestructor<InstanceType>` constructs static singletons in aligned storage and intentionally never destroys them;
- `RoundUp<N>()` in `util/crc32c.cc` uses a non-type template parameter for alignment;
- tests and helper utilities use templates sparingly.

This is not template metaprogramming-heavy code. The sophistication is in using templates only where zero-overhead genericity matters.

### Smart Pointers and Ownership

Production LevelDB code is mostly raw-pointer based. Reasons:

- much of the code predates C++11 idioms;
- public API is C-compatible in spirit and has explicit `delete` contracts;
- iterators and cache handles need custom cleanup semantics;
- intrusive ref-counting is used for versions, memtables, and cache handles.

There is limited `std::unique_ptr` usage in tests. The production ownership model is instead:

- "caller deletes" for returned DBs, iterators, caches, policies;
- intrusive `Ref()`/`Unref()` for `MemTable`, `Version`, helper file state;
- `Iterator::RegisterCleanup()` for attached lifetime cleanup;
- destructors for internal aggregates such as `Table::Rep`, `Arena`, and `Status`.

This style demands discipline, but the code documents ownership close to the API boundary.

### Move Semantics

Move semantics appear where they give clear value:

- `Status` defines move constructor and move assignment. Moving swaps/transfers the internal `state_` pointer.
- platform environment file wrappers in `util/env_posix.cc` and `util/env_windows.cc` use `std::move(filename)` and move-only file-handle wrappers.
- some error-path code moves `Status` in tests.

The core database objects disable copying and generally avoid moving because their identity and lifetime are tied to files, locks, background work, and intrusive references.

### Thread Safety, Atomics, and Memory Ordering

Thread-safety is designed around coarse-grained DB state protection and narrow lock-free read structures.

Major mechanisms:

- `DBImpl::mutex_` protects persistent DB state, memtable rotation, writer queue, snapshots, versions, and compaction state;
- `DBImpl::writers_` serializes writes and enables group commit;
- `background_work_finished_signal_` coordinates writers and background compaction;
- `shutting_down_` is `std::atomic<bool>` with acquire/release checks across background threads;
- `has_imm_` lets compaction loops cheaply notice immutable memtable work;
- skip-list next pointers use acquire/release atomics;
- arena memory usage is an atomic relaxed counter, while allocation itself is externally synchronized by write serialization;
- LRU cache shards each have their own mutex;
- `port/thread_annotations.h` provides Clang thread-safety annotations when supported.

The code distinguishes where it needs ordering and where relaxed atomics are enough. For example, skip-list publication needs release/acquire because readers must see initialized nodes. `max_height_` can be relaxed because stale values are safe.

### `Status`: Error Handling Without Exceptions

`include/leveldb/status.h` represents either OK or an error code plus message. OK is zero allocation:

```cpp
Status() noexcept : state_(nullptr) {}
bool ok() const { return state_ == nullptr; }
```

Error state stores:

```text
4 bytes message length
1 byte code
message bytes
```

This gives:

- explicit error propagation;
- no exceptions across API boundaries;
- cheap success;
- stable behavior in low-level code;
- move support to avoid copies.

The design is similar in spirit to modern `absl::Status`, but smaller.

### Comparator Interface

`Comparator` defines total ordering and two key-shortening hooks:

- `Compare(a, b)`;
- `Name()`;
- `FindShortestSeparator(start, limit)`;
- `FindShortSuccessor(key)`.

The comparator name is part of persistent compatibility: opening a DB with a comparator that orders keys differently can corrupt interpretation of SSTables, so LevelDB records and checks comparator identity.

Internally, `InternalKeyComparator` wraps the user comparator and breaks ties by decreasing sequence number. That one design choice makes "newest visible version first" fall naturally out of ordinary sorted lookup.

### Cache: Sharded LRU

`util/cache.cc` implements an LRU cache with:

- 16 shards (`kNumShardBits = 4`);
- custom hash table;
- intrusive `LRUHandle`;
- reference counts that include cache and client references;
- two lists per shard: `in_use_` and `lru_`;
- user-supplied deleter callbacks.

The split between `in_use_` and `lru_` is subtle. An entry currently held by a reader cannot be evicted, even if it has been removed from the hash table. It is actually destroyed only when its refcount reaches zero.

LevelDB uses caches in two places:

- `TableCache` caches opened `TableAndFile` objects by file number;
- `Table::BlockReader()` caches data blocks by table cache id plus block offset.

### Filter Policy

`FilterPolicy` is pluggable and stored by name. If the filter encoding changes incompatibly, policy `Name()` must change so old filters are not misread. This mirrors comparator naming and is another persistent-format compatibility mechanism.

The warning in `filter_policy.h` is important: if a custom comparator ignores part of a key, the filter policy must ignore the same part. Otherwise the filter can produce false negatives, which would be catastrophic.

### Coding Conventions

The code mostly follows Google C++ style:

- two-space indentation;
- include guards;
- small headers with clear comments;
- deleted copy constructors/assignments for owning or identity types;
- `namespace leveldb`;
- explicit ownership comments;
- `assert()` for internal invariants;
- no exceptions;
- compact classes and short functions where possible.

Notable style characteristics:

- comments often state concurrency and ownership contracts, not just what code does;
- hot-path code uses simple loops and pointer arithmetic instead of abstraction-heavy libraries;
- public interfaces are conservative and stable;
- tests exercise behavior through public APIs and fault-injection environments.

## 5. Engineering Design Philosophy

### Separation of Interface and Implementation

`include/leveldb/` defines the stable API. Implementation lives in `db/`, `table/`, and `util/`. This lets users depend on a small conceptual model:

```cpp
DB::Open(options, path, &db);
db->Put(...);
db->Get(...);
delete db;
```

while the engine retains freedom to change internal compaction, file metadata, caches, and iterators.

### Abstract the Platform

`Env` is one of LevelDB's strongest design choices. The database does not directly call POSIX APIs from core logic; it asks an environment to create files, lock files, schedule work, sleep, log, and report time. `port/` then handles mutexes, compression availability, CRC acceleration, and platform-specific details.

This buys portability, testability, and controlled dependency injection.

### Composition Over Inheritance

LevelDB does use inheritance for narrow polymorphic interfaces, but complex behavior is mostly composed:

- DB iteration = memtable iterator + immutable iterator + table iterators + merging iterator + DB iterator adapter;
- SSTable = data blocks + index block + metaindex block + filter block + footer;
- read path = Version metadata + TableCache + Table + Block + FilterPolicy;
- write path = WriteBatch + WAL writer + MemTable + VersionEdit.

Inheritance is reserved for strategy boundaries. Internal algorithms compose simple objects.

### Small, Focused Classes

Examples:

- `Slice`: non-owning byte view;
- `Status`: operation result;
- `LookupKey`: stack/local encoded lookup key;
- `BlockBuilder`: one block;
- `TableBuilder`: one table file;
- `TableCache`: opened table cache;
- `VersionEdit`: metadata delta;
- `MutexLock`: scoped lock.

Each class has a narrow reason to exist.

### Easy to Use Correctly, Hard to Use Incorrectly

The API exposes safe defaults:

- bytewise comparator by default;
- Snappy compression by default when available, with fallback to uncompressed when not beneficial or unsupported;
- block cache automatically created if none is supplied;
- `Status` return values instead of hidden exceptions;
- snapshots are explicit objects with explicit release;
- `WriteOptions::sync` clearly documents crash semantics.

The code also encodes danger in names and comments: `DestroyDB` says "Be very careful"; comparator/filter compatibility warnings are explicit; iterator `Slice` lifetimes are documented.

### Performance-Critical Paths

Optimizations are concentrated where they matter:

- `Slice` avoids copies;
- `LookupKey` uses a 200-byte stack buffer for common short keys;
- memtable insertion uses arena allocation;
- skip-list readers avoid locks;
- writes are group-committed;
- block keys are prefix-compressed;
- block restart points enable binary seek;
- index keys are shortened;
- Bloom filters avoid unnecessary block reads;
- block cache avoids repeated decompression/read work;
- table cache avoids repeatedly opening/parsing SSTables;
- varint and fixed encoding functions are hand-tuned;
- CRC has hardware acceleration hooks.

### Testing Philosophy

The repository contains focused tests for:

- DB behavior (`db/db_test.cc`);
- recovery and corruption;
- log format;
- skip list;
- version set and version edit;
- table/block/filter behavior;
- cache;
- arena;
- Bloom filter;
- coding;
- CRC32C;
- env implementations;
- historical issues in `issues/`.

Tests use custom `Env` implementations and wrappers to inject filesystem errors, sync failures, read counts, and in-memory storage. This is exactly what the `Env` abstraction enables.

In this checkout, running tests is blocked until `third_party/googletest` is populated.

### Error Handling Strategy

LevelDB consistently returns `Status`, records background errors in `DBImpl::bg_error_`, and propagates persistent errors to future writes where ambiguity would be dangerous. For example, if syncing a WAL record fails, `DBImpl::Write()` records a background error because it cannot know whether the record will appear during recovery.

This is conservative systems engineering: fail closed when persistence state is ambiguous.

### Backward Compatibility

Persistent compatibility appears in several places:

- DB version constants in `include/leveldb/db.h`;
- fixed enum values for `CompressionType` and `ValueType`;
- comparator `Name()` check;
- filter policy `Name()` check;
- `.sst` and older `.ldb` filename fallback in `TableCache::FindTable()`;
- `VersionEdit` and MANIFEST logs for metadata evolution;
- footer/metaindex/index separation in table format, allowing extension.

The code comments repeatedly warn not to change values embedded in on-disk data.

### Simplicity vs Performance

LevelDB is not simple because it avoids hard problems; it is simple because it solves each hard problem with the smallest sufficient mechanism:

- one writer queue rather than a highly concurrent write engine;
- one background compaction scheduling path;
- simple monotonic arena rather than general allocator machinery;
- no exceptions;
- no general transaction system beyond `WriteBatch`;
- no SQL/parser/query layer;
- few dependencies.

The result is performance through locality, sorted structure, batching, and carefully chosen abstractions.

## 6. Key File Walkthroughs

### `db/db_impl.cc`

This is the heart of the database.

Construction initializes the sanitized options, internal comparator, internal filter policy, table cache, writer batch buffer, version set, and background-state flags. The destructor sets `shutting_down_`, waits for scheduled compaction to finish, unlocks the DB file, unreferences memtables, and deletes owned resources.

`DB::Open()`:

1. Allocates `DBImpl`.
2. Locks `impl->mutex_`.
3. Calls `Recover()` to load/create the database and replay logs.
4. Creates a new log and memtable if recovery did not produce one.
5. Applies a `VersionEdit` if needed.
6. Deletes obsolete files.
7. Unlocks and returns the DB pointer.

`DBImpl::Write()`:

1. Constructs a stack `Writer` containing batch, sync flag, condition variable, status, and done flag.
2. Pushes it into `writers_`.
3. Waits until it is at the front.
4. Calls `MakeRoomForWrite()`, which may delay, wait, or rotate memtables.
5. Builds a group commit batch through `BuildBatchGroup()`.
6. Assigns sequence numbers.
7. Unlocks the mutex while appending to WAL, syncing if requested, and inserting into memtable.
8. Re-locks, records sequence advancement, marks grouped writers done, and signals the next writer.

The important invariant is that a writer at the queue front is responsible for both WAL append and memtable insertion, so concurrent writes cannot interleave those phases incorrectly.

`DBImpl::MakeRoomForWrite()`:

1. Returns existing background error if present.
2. Sleeps 1 ms once if Level 0 file count is near slowdown threshold.
3. Returns if current memtable has room.
4. Waits if immutable memtable is still being compacted.
5. Waits if too many Level 0 files exist.
6. Otherwise creates a new log file, closes the old one, moves `mem_` to `imm_`, creates a new `MemTable`, and schedules compaction.

This function is where write backpressure connects to compaction health.

`DBImpl::Get()`:

1. Chooses snapshot sequence from `ReadOptions` or current last sequence.
2. Ref-counts `mem_`, `imm_`, and current `Version`.
3. Unlocks while searching.
4. Searches mutable memtable, immutable memtable, then versioned tables.
5. Re-locks, possibly schedules seek-triggered compaction, unreferences selected structures.

The unlock around potentially slow reads is essential. Ref-counting makes it safe.

`DBImpl::NewInternalIterator()`:

1. Locks and captures latest sequence.
2. Creates memtable and immutable iterators.
3. Asks current version to add table iterators.
4. Wraps them in a merging iterator.
5. Registers cleanup that unreferences memtables and version.

This is a clean example of lifetime management without smart pointers.

`DBImpl::BackgroundCompaction()`:

1. Flushes `imm_` first if present.
2. Otherwise chooses manual or automatic compaction.
3. Performs trivial file move when possible.
4. Otherwise allocates `CompactionState` and calls `DoCompactionWork()`.
5. Releases inputs, removes obsolete files, updates manual compaction progress.

`DBImpl::DoCompactionWork()`:

1. Builds an input iterator over compaction inputs.
2. Releases the DB mutex during data processing.
3. Periodically prioritizes immutable memtable compaction.
4. Parses internal keys.
5. Drops hidden versions and obsolete tombstones.
6. Opens output SSTables lazily.
7. Adds surviving key/value pairs to `TableBuilder`.
8. Splits output files by size and grandparent-overlap rules.
9. Re-locks and installs a `VersionEdit`.

This function embodies the LSM invariant: new sorted files replace old sorted files atomically at the metadata level.

### `db/skiplist.h`

The file starts with an unusually precise concurrency contract:

- writes require external synchronization;
- reads require only object lifetime stability;
- nodes are never deleted until skip-list destruction;
- node payload is immutable after publication;
- release stores publish nodes and acquire loads observe them.

The `Node` type stores:

```cpp
Key const key;
std::atomic<Node*> next_[1];
```

The `next_[1]` member is a C-style flexible-array idiom. `NewNode()` allocates enough arena memory for the desired height and constructs `Node` with placement new.

Search:

- `FindGreaterOrEqual()` starts from the top level and moves right while next key is still less than target; otherwise it drops one level.
- `FindLessThan()` performs a similar search for predecessor.
- `FindLast()` walks to the rightmost node at each level.

Insert:

1. Finds predecessors at every level.
2. Picks a random height.
3. Raises `max_height_` if needed.
4. Allocates a node.
5. Initializes each new node pointer with relaxed stores.
6. Publishes the node by storing it into predecessor links with release stores.

The result is a structure optimized for LevelDB's exact need: one synchronized writer path, many concurrent readers, no per-node reclamation.

### `table/table_builder.cc`

`TableBuilder` writes the physical SSTable format.

Constructor:

- creates a `Rep`;
- initializes data and index block builders;
- creates a filter block builder if a filter policy exists;
- starts the first filter block at offset 0.

`Add(key, value)`:

1. Asserts keys arrive in strictly increasing comparator order.
2. If a previous data block was flushed, emits its pending index entry using a shortened separator between previous last key and current key.
3. Adds the key to the filter block.
4. Adds key/value to the current `BlockBuilder`.
5. Flushes if estimated block size reaches `options.block_size`.

`Flush()`:

1. Writes the current data block using `WriteBlock()`.
2. Records the resulting `BlockHandle`.
3. Marks an index entry as pending.
4. Flushes the writable file.
5. Starts a new filter range at the current file offset.

`WriteBlock()`:

1. Finalizes the block.
2. Attempts configured compression.
3. Keeps compressed data only if it saves at least 12.5%.
4. Calls `WriteRawBlock()`.
5. Resets the block builder.

`WriteRawBlock()`:

1. Appends block contents.
2. Appends one byte of compression type.
3. Computes masked CRC over block contents plus type byte.
4. Appends CRC trailer.
5. Advances file offset.

`Finish()` writes, in order:

1. pending data block;
2. filter block;
3. metaindex block;
4. index block;
5. footer.

The destructor asserts `Finish()` or `Abandon()` was called. That assertion catches API misuse during development.

### `util/arena.h` and `util/arena.cc`

The arena is intentionally minimal.

State:

```cpp
char* alloc_ptr_;
size_t alloc_bytes_remaining_;
std::vector<char*> blocks_;
std::atomic<size_t> memory_usage_;
```

`Allocate(bytes)` is inline for speed:

1. Assert nonzero size.
2. If current block has enough room, return current pointer and bump.
3. Otherwise call `AllocateFallback()`.

`AllocateFallback(bytes)`:

- for large requests, allocate exactly that size as a separate block;
- for normal requests, allocate a fresh 4096-byte block and carve from it.

`AllocateAligned(bytes)`:

1. Computes alignment as max pointer size and 8.
2. Computes slop from the current pointer.
3. Uses current block if possible.
4. Otherwise falls back to a new block.

`~Arena()` deletes every block in `blocks_`.

The arena does not call destructors for allocated objects. That is appropriate because LevelDB uses it for trivial-ish skip-list node storage whose lifetime ends wholesale with the memtable.

### `db/memtable.cc`

`MemTable` is the write buffer's sorted in-memory representation.

Construction:

- stores an `InternalKeyComparator`;
- initializes reference count to zero;
- constructs a skip list using the arena.

`Add()` encodes a single internal record and inserts a pointer to it into the skip list. The encoded internal key includes the user key plus sequence/type tag. The comparator decodes length-prefixed keys and compares them by internal-key order.

`Get()`:

1. Builds the memtable lookup slice from `LookupKey`.
2. Seeks the skip list.
3. Checks user-key equality.
4. Decodes the tag.
5. Returns value, deletion, or miss.

This file shows why internal-key ordering is powerful: one seek finds the newest visible update for a user key.

## 7. Summary: What Makes LevelDB Great C++

LevelDB is great C++ because it aligns language mechanisms with system invariants:

- non-owning `Slice` makes byte movement explicit and cheap;
- `Status` makes failure explicit and cheap on success;
- `Env`, `Comparator`, `FilterPolicy`, and `Cache` isolate policy from mechanism;
- arena allocation matches memtable lifetime;
- skip-list memory ordering matches the single-writer/many-reader design;
- iterators compose sorted sources without exposing internal storage layout;
- `VersionSet` gives copy-on-write metadata snapshots for concurrent reads and compaction;
- `TableBuilder` and `BlockBuilder` turn a sorted stream into a compact searchable file format;
- compaction logic preserves snapshot semantics while reclaiming obsolete history;
- the public API is small, stable, and heavily documented around ownership and synchronization.

The dominant engineering philosophy is precise minimalism. LevelDB does not hide complexity behind large frameworks. It reduces each requirement to a small mechanism: sorted immutable files, append-only logs, versioned metadata, explicit status returns, carefully scoped locks, and simple abstractions at real seams such as platform I/O and key ordering. That is why the code remains readable while implementing a serious storage engine.
