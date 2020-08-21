# 基础组件：Store 和 Span

Store 用于描述由多个物理存储设备构成的集合，Span 用于描述一个物理设备上一个或多个区域。

Store 被 MultiCache 和 Cache 子系统作为访问物理存储设备的中介层，通过 Store 与一个或多个物理设备或文件系统上的文件建立联系。

```
Store::disk[0]                     Store::disk[1]         ......      Store::disk[n]
    |                                  |                                  |
    |   Disk offset                    |   File offset                    |   Disk offset
    |         [0] +------------+       |         [0] +------------+       |         [0] +------------+
    |             |  /dev/sda  |       |             |  /opt/db1  |       |             |  /dev/sdX  |
    V             |            |       V             |            |       V             |            |
 Span::offset ----> +--------+ |    Span::offset ----> +--------+ |    Span::offset ----> +--------+ |
    |             | |        | |       |             | |        | |       |             | | Span x | |
    |             | | Span 1 | |       |             | |        | |       |   end() ----> +--------+ |
   next           | |        | |      next           | | Span 3 | |       V             |            |
    |   end() ----> +--------+ |       |             | |        | |    Span::offset ----> +--------+ |
    V             |            |       |             | |        | |       |             | | Span y | |
 Span::offset ----> +--------+ |       |   end() ----> +--------+ |       |   end() ----> +--------+ |
    |             | |        | |       V             +------------+       V             |            |
    |             | | Span 2 | |      NULL                             Span::offset ----> +--------+ |
   next           | |        | |                                          |             | |        | |
    |   end() ----> +--------+ |                                          |             | | Span z | |
    V             +------------+                                          |             | |        | |
   NULL                                                                   |   end() ----> +--------+ |
                                                                          V             +------------+
** n = Store::n_disks - 1                                                 NULL
```

## Span

Span 被用来指向一个具体的物理设备上连续的存储空间，或文件系统上的文件或目录。

### 定义

```
//
// A Store is a place to store data.
// Those on the same disk should be in a linked list.
//
struct Span {
  // 以 STORE_BLOCK_SIZE 的字节数为一个 BLOCK，表示当前 Span 包含了多少个 BLOCK
  int64_t blocks; // in STORE_BLOCK_SIZE blocks
  // 对于 MultiCache：
  //  - 当 file_pathname 为 true 时，保留该文件头部一定 BLOCK，不用于该 Span 的数据存储
  //  - 当 file_pathname 为 false 时，该值必须为 0，否则会在调用 store_verify 方法时触发 assert
  // 注释中的 in bytes 是不正确的（由 commit 4747fe8f7c1c4bf6db4512f9d8ba0322b570dc8a 引入）
  int64_t offset; // used only if (file == true); in bytes
  // 底层硬件设备的扇区大小，默认为 DEFAULT_HW_SECTOR_SIZE (512)
  unsigned hw_sector_size;
  // 当 file_pathname 为 true 时，将文件与底层硬件的扇区进行对齐，以字节为单位，
  // 当使用 LVM-JBOD 或 LVM-RAID 时该值可能不为 0，具体可以查看 ink_file_get_geometry 方法
  // 但是该值通常是 512 字节的整数倍，例如：
  //   - 在 4KB 扇区对齐的硬盘上，当第一个 Span 的长度如果不是 4KB 的整数倍，
  //   - 那么会导致第二个 Span 的起始位置与物理扇区没有与物理扇区对齐。
  // 例如在 LVM 管理多个硬盘时，可能会将 512B 与 4KB 扇区对齐的硬盘混合在一起。
  unsigned alignment;
  // disk_id 用来保存底层设备的唯一描述信息，由两个 64 bits 的整形数组成（其值可参考 man fstat）
  // - 对于块和字符设备，disk_id[0] = 0，disk_id[1] = stat.st_rdev，
  // - 对于文件和目录，disk_id[0] = stat.st_dev，disk_id[1] = stat.st_ino，
  span_diskid_t disk_id;
  // 指定 volume id，通过 storage.config 的 volume 参数指定
  int forced_volume_num; ///< Force span in to specific volume.
private:
  // 是否支持 mmap，通过 is_mmapable() 和 set_mmapable() 方法读写该私有成员
  bool is_mmapable_internal;

public:
  // Span 的存储是文件还是目录，注意：文件可能是 /dev/sda 也可能是 /var/cache/cache.db
  // 如果是文件，会自动判断当前文件长度作为 Span 的大小
  // 如果是目录，则会按照 storage.config 里的配置，在目录下创建指定大小的数据文件
  bool file_pathname; // the pathname is a file
  // v- used as a magic location for copy constructor.
  // we memcpy everything before this member and do explicit assignment for the rest.
  // 用于保存 Span 物理存储的文件名或目录名
  // 拷贝构造函数会用该成员的偏移量作为 memcpy 的长度，详见拷贝构造函数
  ats_scoped_str pathname;
  // 如果在 storage.config 里指定了 id=string 的参数，则将其存储在该成员里
  // 在进行一致性哈希计算时使用该值替代 pathname 参与哈希计算
  ats_scoped_str hash_base_string; ///< Used to seed the stripe assignment hash.
  // 用于将多个位于同一个物理设备的 Span 连接成链表
  SLINK(Span, link);

  bool
  is_mmapable() const
  {
    return is_mmapable_internal;
  }
  void
  set_mmapable(bool s)
  {
    is_mmapable_internal = s;
  }
  // 取得当前 Span 的大小，不包含 Span chain 上其它 Span，以字节为单位
  int64_t
  size() const
  {
    return blocks * STORE_BLOCK_SIZE;
  }

  // 取得当前 Span chain 上所有 Span 的 BLOCK 数量的总和
  int64_t
  total_blocks() const
  {
    if (link.next) {
      return blocks + link.next->total_blocks();
    } else {
      return blocks;
    }
  }

  // 获取 Span chain 上的第 i 个 Span
  Span *
  nth(unsigned i)
  {
    Span *x = this;
    while (x && i--)
      x = x->link.next;
    return x;
  }

  // 计算当前 Span chain 上有多少个 Span
  unsigned
  paths() const
  {
    int i = 0;
    for (const Span *x = this; x; i++, x = x->link.next)
      ;
    return i;
  }

  // 读写 Span 的配置信息，被 Store::write 和 Store::read 方法调用，请参考 Store 中的说明
  int write(int fd) const;
  int read(int fd);

  /// Duplicate this span and all chained spans.
  // 复制当前 Span 以及链表上所有的 Span 对象，被 Store::dup 方法调用
  Span *dup();
  // 返回当前 Span 的尾部位置（以 BLOCK 为单位）
  int64_t
  end() const
  {
    return offset + blocks;
  }

  // 初始化 Span 对象
  // 参数 n 指向物理设备的路径或文件系统上的目录或文件，
  // 参数 size 为 Span 对象的大小，以字节为单位。
  const char *init(const char *n, int64_t size);

  // 0 on success -1 on failure
  // 获取数据文件的文件名
  // 当 file_pathname 为 true 时，将 pathname 直接填充到 buf 里，并返回文件名的长度，
  // 当 file_pathname 为 false 时，将 pathname 与 filename 拼接后填充到 buf 里，并返回拼接后的文件名长度，
  // 参数 buflen 用于指定 buf 的空间大小，如果空间不足，则返回 -1。
  int path(char *filename,         // for non-file, the filename in the director
           int64_t *offset,        // for file, start offset (unsupported)
           char *buf, int buflen); // where to store the path

  /// Set the hash seed string.
  void hash_base_string_set(char const *s);
  /// Set the volume number.
  void volume_number_set(int n);

  // 构造函数
  Span()
    : blocks(0), offset(0), hw_sector_size(DEFAULT_HW_SECTOR_SIZE), alignment(0), forced_volume_num(-1),
      is_mmapable_internal(false), file_pathname(false)
  {
    disk_id[0] = disk_id[1] = 0;
  }

  /// Copy constructor.
  /// @internal Prior to this implementation handling the char* pointers was done manual
  /// at every call site. We also need this because we have ats_scoped_str members.
  // 拷贝构造函数，主要被 dup 方法调用
  Span(Span const &that)
  {
    memcpy(this, &that, reinterpret_cast<intptr_t>(&(static_cast<Span *>(0)->pathname)));
    if (that.pathname)
      pathname = ats_strdup(that.pathname);
    if (that.hash_base_string)
      hash_base_string = ats_strdup(that.hash_base_string);
    link.next = NULL;
  }

  // 析构函数，递归析构 Span 链表
  ~Span();

  // 返回与 Span 错误代码相关的 debug 信息，主要在 Span::init 方法里调用
  static const char *errorstr(span_error_t serr);
};
```


### 方法

#### Span::init(const char *path, int64_t size)

```
const char *
Span::init(const char *path, int64_t size)
{
  struct stat sbuf;
  struct statvfs vbuf;
  span_error_t serr;
  ink_device_geometry geometry;

  // 判断文件或目录是否存在，不存在则跳转到错误处理
  ats_scoped_fd fd(socketManager.open(path, O_RDONLY));
  if (fd < 0) {
    serr = make_span_error(errno);
    Warning("unable to open '%s': %s", path, strerror(errno));
    goto fail;
  }

  // 获取文件或目录的信息
  if (fstat(fd, &sbuf) == -1) {
    serr = make_span_error(errno);
    Warning("unable to stat '%s': %s (%d)", path, strerror(errno), errno);
    goto fail;
  }

  // 获取文件或目录所在的磁盘分区的信息
  if (fstatvfs(fd, &vbuf) == -1) {
    serr = make_span_error(errno);
    Warning("unable to statvfs '%s': %s (%d)", path, strerror(errno), errno);
    goto fail;
  }

  // Directories require an explicit size parameter. For device nodes and files, we use
  // the existing size.
  // 如果是目录，必须指定 Span 的长度；如果是文件则，Span 长度是可选的。
  if (S_ISDIR(sbuf.st_mode)) {
    if (size <= 0) {
      Warning("cache %s '%s' requires a size > 0", span_file_typename(sbuf.st_mode), path);
      serr = SPAN_ERROR_MISSING_SIZE;
      goto fail;
    }
  }

  // Should regular files use the IO size (vbuf.f_bsize), or the
  // fundamental filesystem block size (vbuf.f_frsize)? That depends
  // on whether we are using that block size for performance or for
  // reliability.
  // 只看与文件类型相关的 bit 位，其值可能为：S_IFSOCK，S_IFLNK，S_IFREG，S_IFBLK，S_IFDIR，S_IFCHR，S_IFIFO
  switch (sbuf.st_mode & S_IFMT) {
  case S_IFBLK:
  case S_IFCHR: // 块设备或字符设备
    // 支持字符设备是因为 Linux 自身提供了 raw character device （详情请查阅 man 8 raw）
#if defined(linux)
    if (major(sbuf.st_rdev) == RAW_MAJOR && minor(sbuf.st_rdev) == 0) {
      Warning("cache %s '%s' is the raw device control interface", span_file_typename(sbuf.st_mode), path);
      serr = SPAN_ERROR_UNSUPPORTED_DEVTYPE;
      goto fail;
    }
#endif

    // 获取该文件或目录所在物理设备的硬件信息
    if (!ink_file_get_geometry(fd, geometry)) {
      serr = make_span_error(errno);

      if (errno == ENOTSUP) {
        Warning("failed to query disk geometry for '%s', no raw device support", path);
      } else {
        Warning("failed to query disk geometry for '%s', %s (%d)", path, strerror(errno), errno);
      }

      goto fail;
    }

    this->disk_id[0] = 0;
    this->disk_id[1] = sbuf.st_rdev;
    this->file_pathname = true;
    // 物理扇区的大小，可能为 512 或 4096 字节
    this->hw_sector_size = geometry.blocksz;
    // 物理扇区对齐值，通常为 512 字节的整数倍
    this->alignment = geometry.alignsz;
    // 对于块设备，总是以该设备的总容量作为该 Span 的空间大小
    // 将 Span 空间的大小换算为 BLOCK 的数量（默认 BLOCK 为 8KB），这里会向下取整
    this->blocks = geometry.totalsz / STORE_BLOCK_SIZE;

    break;

  case S_IFDIR: // 目录
    // 当存储位置为目录时，必须指定 Span 的空间大小，此处需要校验该目录所在的磁盘分区有足够的可分配空间
    if ((int64_t)(vbuf.f_frsize * vbuf.f_bavail) < size) {
      // 如果空间不足时，仅给出警告，稍后在 MultiCache::open 时再报错
      Warning("not enough free space for cache %s '%s'", span_file_typename(sbuf.st_mode), path);
      // Just warn for now; let the cache open fail later.
    }

    // The cache initialization code in Cache.cc takes care of creating the actual cache file, naming it and making
    // it the right size based on the "file_pathname" flag. That's something that we should clean up in the future.
    this->file_pathname = false;

    this->disk_id[0] = sbuf.st_dev;
    this->disk_id[1] = sbuf.st_ino;
    // 文件系统的 I/O 块大小
    this->hw_sector_size = vbuf.f_bsize;
    // 对于目录，该值固定为 0
    this->alignment = 0;
    // 将 Span 空间的大小换算为 BLOCK 的数量（默认 BLOCK 为 8KB），这里会向下取整
    // 这里使用了传入的参数 size 计算 Span 的空间，因此当配置修改后，会直接导致 Span 的容量发生变化
    this->blocks = size / STORE_BLOCK_SIZE;
    break;

  case S_IFREG: // 文件
    // 如果给定的 Span 长度大于 0 并且大于该文件的实际长度，则需要判断当前分区的剩余空间是否能够完成文件长度的扩展（不支持？）
    if (size > 0 && sbuf.st_size < size) {
      int64_t needed = size - sbuf.st_size;
      if ((int64_t)(vbuf.f_frsize * vbuf.f_bavail) < needed) {
        // 如果空间不足时，仅给出警告，稍后在 MultiCache::open 时再报错
        Warning("not enough free space for cache %s '%s'", span_file_typename(sbuf.st_mode), path);
        // Just warn for now; let the cache open fail later.
      }
    }

    this->disk_id[0] = sbuf.st_dev;
    this->disk_id[1] = sbuf.st_ino;
    this->file_pathname = true;
    // 文件系统的 I/O 块大小
    this->hw_sector_size = vbuf.f_bsize;
    // 对于普通文件，该值固定为 0
    this->alignment = 0;
    // 将 Span 空间的大小换算为 BLOCK 的数量（默认 BLOCK 为 8KB），这里会向下取整
    // 使用文件当前长度作为 Span 空间的大小，看上去目前还不支持根据配置文件进行调整数据文件大小的功能，这里直接忽略掉了 size
    this->blocks = sbuf.st_size / STORE_BLOCK_SIZE;

    break;

  default:
    // 不支持其它类型：S_IFSOCK，S_IFLNK，S_IFIFO
    serr = SPAN_ERROR_UNSUPPORTED_DEVTYPE;
    goto fail;
  }

  // The actual size of a span always trumps the configured size.
  // 如果指定的 Span 长度小于文件长度或者块设备的长度，则以指定的长度为准，重新计算 Span 的 BLOCK 数量
  if (size > 0 && this->size() != size) {
    int64_t newsz = MIN(size, this->size());

    Note("cache %s '%s' is %" PRId64 " bytes, but the configured size is %" PRId64 " bytes, using the minimum",
         span_file_typename(sbuf.st_mode), path, this->size(), size);

    this->blocks = newsz / STORE_BLOCK_SIZE;
  }

  // A directory span means we will end up with a file, otherwise, we get what we asked for.
  // 根据文件或目录的设备类型设置 Span 是否可以进行 mmap 操作
  // 因为当指定一个目录时，会在该目录下创建一个指定大小的文件，因此这里将目录类型转为文件类型
  this->set_mmapable(ink_file_is_mmappable(S_ISDIR(sbuf.st_mode) ? (mode_t)S_IFREG : sbuf.st_mode));
  this->pathname = ats_strdup(path);

  Debug("cache_init", "initialized span '%s'", this->pathname.get());
  Debug("cache_init", "hw_sector_size=%d, size=%" PRId64 ", blocks=%" PRId64 ", disk_id=%" PRId64 "/%" PRId64 ", file_pathname=%d",
        this->hw_sector_size, this->size(), this->blocks, this->disk_id[0], this->disk_id[1], this->file_pathname);

  // 成功返回 NULL，失败返回错误描述信息的字符串
  return NULL;

fail:
  return Span::errorstr(serr);
}
```

## Store

Store 内包含了多个 Span 对象，通过 Store::sort 方法位于相同物理设备上的 Span 被连接成链表。

### 定义

```
struct Store {
  //
  // Public Interface
  // Thread-safe operations
  //

  // spread evenly on all disks
  // Store 的空间分配是按照 STORE_BLOCK_SIZE (在 I_Store.h 中定义为 8192) 为单位进行分配的。
  // spread_alloc 方法从 Store 中分配指定的数量的 BLOCK，这些 BLOCK 将会平均分配到当前 Store 中所有的 Span 上。
  void spread_alloc(Store &s, unsigned int blocks, bool mmapable = true);
  // alloc 方法是按照顺序从 Store 中的第一个磁盘的第一个 Span 上开始分配指定数量的 BLOCK
  // 如果第一个 Span 不足以完成分配，则继续在下一个 Span 上分配，
  // 如果第一个磁盘不足以完成分配，则继续在下一个磁盘上分配，
  // 当 only_one 参数为 true 时，则只在第一个可用的 Span 上进行一次分配，实际分配的 BLOCK 数量可能小于 blocks 指定的值；
  // 当 mmapable 参数为 true 时，则只在可以 mmap 的 Span 上进行分配。 
  void alloc(Store &s, unsigned int blocks, bool only_one = false, bool mmapable = true);

  // 调用 alloc 方法时，only_one 为 true
  // 因此只会在 Store 里创建一个 Span，并且将该 Span 返回
  Span *
  alloc_one(unsigned int blocks, bool mmapable)
  {
    Store s;
    alloc(s, blocks, true, mmapable);
    if (s.n_disks) {
      Span *t = s.disk[0];
      s.disk[0] = NULL;
      return t;
    }

    return NULL;
  }
  
  // try to allocate, return (s == gotten, diff == not gotten)
  void try_realloc(Store &s, Store &diff);

  // free back the contents of a store.
  // must have been JUST allocated (no intervening allocs/frees)
  // 释放已经分配的 Store 空间，传入的 Store &s 内的 Span 必须是从当前 Store 对象分配出去的
  void free(Store &s);
  // 将 Span 添加到当前的 Store 里的 disk 指针数组里
  void add(Span *s);
  // 将指定的 Store 内所有的 Span 全部移动到当前的 Store
  void add(Store &s);
  // 将当前 Store 内的 Span 信息复制一份存入指定的 Store
  void dup(Store &s);
  // sort 方法遍历 disk 指针数组，根据 filename 判断 Span 是否位于同一个设备、磁盘或文件，
  // 如果相同，那么就讲这些 Span 连接成链表，在排序完成后，每个 disk 指针数组都指向一个 Span 的链表，
  // 每个链表的 Span 对象都属于同一个设备、磁盘或文件。
  // 每当调用了 add 方法之后，必须调用 sort 方法进行排序
  void sort();
  // 扩展 disk 指针数组，以容纳更多的 Span 对象
  void
  extend(unsigned i)
  {
    if (i > n_disks) {
      disk = (Span **)ats_realloc(disk, i * sizeof(Span *));
      for (unsigned j = n_disks; j < i; j++) {
        disk[j] = NULL;
      }
      n_disks = i;
    }
  }

  // Non Thread-safe operations
  // total_blocks 方法用于计算当前 Store 总共有多少个 BLOCK，
  // 它会遍历 disk 指针数组，累加每一个 Span 的 BLOCK 数量。
  unsigned int
  total_blocks(unsigned after = 0) const
  {
    int64_t t = 0;
    for (unsigned i = after; i < n_disks; i++) {
      if (disk[i]) {
        t += disk[i]->total_blocks();
      }
    }
    return (unsigned int)t;
  }
  // 0 on success -1 on failure
  // these operations are NOT thread-safe
  // 方法 wirte 和 read 用于将当前 Store 的信息保存至 MultiCache 描述文件，
  // 或从 MultiCache 描述文件加载信息到当前 Store，MultiCache 描述文件以换行符 \n 分隔，每行一个信息
  // 需要注意的是在调用 Store::write 之前，MultiCacheBase::write_config 已经向文件中写入了一些信息：
  //   第 01 行：可容纳的元素数量（参考 MultiCacheHeader::nominal_elements）
  //   第 02 行：MultiCache 的 bucket 数量（参考 MultiCacheHeader::bucket）
  //   第 03 行：MultiCache Heap 空间的大小，以字节为单位（参考 MultiCacheHeader::heap_size）
  // 然后是由 Store::write 方法写入：
  //   第 04 行：文件名，不含路径，对于 hostdb 来说该文件位于 /var/cache/trafficserver 目录，用来存储 hostdb 的数据区和 heap 区
  //   第 05 行：Store::n_disks 的值，表示有多少个物理设备
  //   从第一个物理设备开始，遍历所有物理设备
  //       第 06 行：该物理设备内有多少个 Span
  //       从该设备的第一个 Span 开始遍历链表，以下由 Span::write 方法写入
  //           第 07 行：该 Span 所在的路径，或用于保存该 Span 的文件名
  //           第 08 行：第一个 Span 有多少个 BLOCK（以 STORE_BLOCK_SIZE 为单位）
  //           第 09 行：说明第 07 行的路径是一个目录（0）还是文件（1）
  //           第 10 行：如果第 09 行是 1 的时候，表示该 Span 位于文件的偏移位置
  //           第 11 行：说明该 Span 的设备是否支持 mmap 映射：不支持（0），支持（1）
  //       下一个 Span（重复第 07 行 ~ 第 11 行）
  //   下一个物理设备（重复第 06 行 ~ 第 11 行）
  int write(int fd, const char *name) const;
  // 同样的，在调用 Store::read 方法之前，MultiCacheBase::read_config 也已经从文件中读取了前 3 行信息
  int read(int fd, char *name);
  // 遍历 Store 内的每一个 Span，将 Span 的内容清空（使用 0x00 填充 自 offset 开始的一个 BLOCK）
  // 注意：目前调用该方法的只有 MultiCacheBase::open 一处，传入 clear_dirs = false
  //   但是当 clear_dirs 为 true 时，会导致 bug，因此调用此方法时，必须传入 clear_dirs = false。
  int clear(char *filename, bool clear_dirs = true);
  // 遍历 disk 指针数组，将指向 NULL 的元素删除，并重新计算 n_disks 的值
  void normalize();
  // 释放所有 disk 指针数组指向的 Span 对象，并释放 disk 指针数组
  void delete_all();
  // 从当前 Store 里删除指定 pathname 的 Span 对象，可能会删除多个 Span 对象
  int remove(char *pathname);
  // 构造函数
  // 初始化 n_disks 为 0，disk 指针数组为 NULL
  Store();
  // 析构函数
  // 调用 delete_all 方法
  ~Store();

  //  成员 n_disks 表示指针数组 disk 的成员数量
  unsigned n_disks;
  // 在 sort 方法调用之前，disk 数组指向每一个 Span 对象，
  // 在 sort 方法调用之后，disk 数组指向 Span 对象的链表，每个存储设备仅有一个 Span 对象的链表
  Span **disk;
  //
  // returns NULL on success
  // if fd >= 0 then on failure it returns an error string
  //            otherwise on failure it returns (char *)-1
  //
  //////////////////// 以下部分将在 Cache 章节进行分析 ////////////////////
  // 仅用于加载配置文件 storage.config
  const char *read_config();
  // 输出每一个 Span 的路径或文件名以及对应的长度，以字节数计算
  int write_config_data(int fd) const;

  /// Additional configuration key values.
  // 以下两个常量在解析 storage.config 时使用，其格式如下（可参考 storage.config 的说明）：
  //    pathname size [id=string] [volume=number]
  // 静态常量 VOLUME_KEY 为字符串 "volume"
  // 调用 Span::volume_number_set 方法将 volume 的配置设置到 Span 上
  static char const VOLUME_KEY[];
  // 静态常量 HASH_BASE_STRING_KEY 为字符串 "id"
  // 调用 Span::hash_base_string_set 方法将 id 的配置设置到 Span 上，在 Vol::init 里将会判断该信息
  static char const HASH_BASE_STRING_KEY[];
};
```

### 方法

#### try\_alloc(Store &target, Span *source, unsigned int start\_blocks, bool one\_only = false)

`try_alloc` 方法用于从指定的 `Span *source` 上分配空间：

- 该空间以 `STORE_BLOCK_SIZE` 为单位，数量为 `start_blocks`。
- `Span *source` 可能指向一个链表，由位于同一个物理存储设备上的多个 Span 存储对象组成。
- 如果一个 Span 上的 BLOCK 数量不足时，会在 Span 链表上继续查找下一个可用的 Span，并继续完成分配；
- 由于每一次分配都会产生一个新的 Span 对象，因此从多个 Span 上分配空间，其结果也是由多个 Span 对象连接为一个链表。
- 最后将得到的 Span 链表存入 `Store &target` 对象内，同时返回实际完成分配的 BLOCK 数量。

参数 `one_only` 为 `true` 表示仅在第一个 Span 对象上进行空间分配，因此有可能无法成功分配所请求的 `start_blocks` 数量。

```
static unsigned int
try_alloc(Store &target, Span *source, unsigned int start_blocks, bool one_only = false)
{
  unsigned int blocks = start_blocks;
  Span *ds = NULL;
  // 未完成分配，并且有可供分配的 Span
  while (source && blocks) {
    // 判断当前 Span 上是否有可供分配的 BLOCK
    // 个人认为应该写为：source->blocks > 0
    if (source->blocks) {
      // 计算可以从当前 Span 上取走所少个 BLOCK，分为两种情况：
      //   1. 如果当前 Span 上剩余可供分配的 BLOCK 不足，则将当前 Span 上剩余的 BLOCK 全部取走，不足部分从下一个 Span 里获取
      //   2. 如果当前 Span 上剩余可供分配的 BLOCK 充足，则只从当前 Span 上取走所需要的 BLOCK 数量
      unsigned int a; // allocated
      if (blocks > source->blocks)
        a = source->blocks;
      else
        a = blocks;
      // 创建 Span 对象用于保存所分配的 BLOCK，该 Span 对象也可能是一个链表
      Span *d = new Span(*source);

      d->blocks = a;
      d->link.next = ds;

      // 由于是从 Span *source 的头部开始分配空间，因此剩余部分的起始位置需要做对应的调整
      // 如果底层存储设备是一个文件，那么就要修正 offset 值，跳过已经分配的空间
      if (d->file_pathname)
        source->offset += a;
      // 对应的，要减少 Span *source 的 BLOCK 数量
      source->blocks -= a;
      // Span *ds 指向已经分配空间的链表表头
      ds = d;
      // 计算剩余需要分配的空间
      blocks -= a;
      // 如果调用者要求仅进行一次分配，则结束分配操作（可能没有完成所请求的容量）
      if (one_only)
        break;
    }
    // 继续在下一个 Span 上进行分配
    source = source->link.next;
  }
  // 成功完成空间分配，将其添加到 Store
  if (ds)
    target.add(ds);
  // 返回实际完成多少个 BLOCK 的分配
  return start_blocks - blocks;
}
```

#### Store::alloc(Store &s, unsigned int blocks, bool one_only, bool mmapable)

`alloc` 方法用于从当前 Store 上分配指定 `blocks` 数量的空间：

- 该空间以 `STORE_BLOCK_SIZE` 为单位，数量为 `blocks`。
- 如果第一个磁盘未能完成指定 `blocks` 数量的分配，会自动尝试在下一个磁盘上进行分配，
- 分配所得到的空间将会存入 `Store &s` 对象。
- 但是该方法不返回任何信息，需要调用 `s.total_blocks()` 确认是否达成分配任务。

参数 `one_only` 为 `true` 表示仅在当前 Store 的第一个磁盘的第一个 Span 对象上进行空间分配，因此有可能无法成功分配所请求的 `blocks` 数量。
参数 `mmapable` 为 `true` 表示仅在支持 mmap 的磁盘上进行空间分配。

```
//
// Stupid grab first availabled space allocator
//
void
Store::alloc(Store &s, unsigned int blocks, bool one_only, bool mmapable)
{
  unsigned int oblocks = blocks;
  for (unsigned i = 0; blocks && i < n_disks; i++) {
    // 如果 mmapable 为 true 则仅在支持 mmap 的磁盘上进行空间分配
    if (!(mmapable && !disk[i]->is_mmapable())) {
      // 调用 try_alloc 方法完成在 Span disk[i] 上的空间分配
      blocks -= try_alloc(s, disk[i], blocks, one_only);
      // 如果 one_only 为 true 则只进行一次空间分配
      if (one_only && oblocks != blocks)
        break;
    }
  }
}
```

通常不建议使用该方法进行空间分配，因为该方法会造成物理磁盘的读写 I/O 负载不均衡。

#### spread_alloc(Store &s, unsigned int blocks, bool mmapable = true)

`spread_alloc` 方法用于从当前 Store 上分配指定 `blocks` 数量的空间：

- 该空间以 `STORE_BLOCK_SIZE` 为单位，数量为 `blocks`。
- `spread_alloc` 与 `alloc` 方法不同，他会将 BLOCK 平均分配到所有磁盘上
- 分配所得到的空间将会存入 `Store &s` 对象。
- 但是该方法同样不返回任何信息，需要调用 `s.total_blocks()` 确认是否达成分配任务。

```
void
Store::spread_alloc(Store &s, unsigned int blocks, bool mmapable)
{
  //
  // Count the eligable disks..
  //
  // 统计支持 mmap 的磁盘数量
  int mmapable_disks = 0;
  for (unsigned k = 0; k < n_disks; k++) {
    if (disk[k]->is_mmapable()) {
      mmapable_disks++;
    }
  }

  // 空间将会被平均分布到 n_disks 个磁盘
  int spread_over = n_disks;
  // 如果 mmapable 为 true，则可用于空间分配的磁盘设置为支持 mmap 的磁盘数量
  if (mmapable) {
    spread_over = mmapable_disks;
  }

  // 可供进行空间分配的磁盘数量为 0 则返回，例如：
  //  - 未向当前 Store 添加磁盘对象和 Span 对象，
  //  - 所有的磁盘都不支持 mmap 功能，等
  if (spread_over == 0) {
    return;
  }

  int disks_left = spread_over;

  // 遍历当前 Store 的所有磁盘对象，
  // 同时对剩余待分配空间 blocks 和剩余可用磁盘数量 disks_left 进行判断
  for (unsigned i = 0; blocks && disks_left && i < n_disks; i++) {
    // 如果 mmapable 为 true 则仅在支持 mmap 的磁盘上进行空间分配
    if (!(mmapable && !disk[i]->is_mmapable())) {
      // 使用剩余待分配空间 blocks 除以剩余可用磁盘数量得到当前磁盘上需要分配的空间 target
      unsigned int target = blocks / disks_left;
      // 由于每一个磁盘的总容量不同，如果在完成本次分配后，剩余的所有磁盘上的空间不足以完成空间分配，
      // 那么就要在当前磁盘上多分配一些空间，通过 total_blocks(i + 1) 计算出后面所有磁盘的可用空间，
      if (blocks - target > total_blocks(i + 1))
        target = blocks - total_blocks(i + 1);
      // 调用 try_alloc 方法在当前 Span disk[i] 上完成空间分配，
      // 并调整剩余待分配空间 blocks 的值和剩余可用磁盘数量 disks_left 的值
      blocks -= try_alloc(s, disk[i], target);
      disks_left--;
    }
  }
}
```

刚方法将所需的空间平均分配到多个物理存储设备，在读写该空间时，对物理存储设备的 I/O 操作也会分布到多个设备，有利于平衡 I/O 操作的负载。

问题：上面在调用 total_blocks(i + 1) 时，没有考虑 mmapable 为 true 的情况，可能会导致潜在的分配问题。

#### Store::try_realloc(Store &s, Store &diff)

`try_realloc` 方法在给定的 `Store &s` 中遍历所有的 Span 对象，并尝试在当前的 Store 里查询是否有与指定的 Span 对象存在空间重叠的 Span 对象；如存在则按照该 Span 的信息在当前 Store 里分配空间，将已经分配的部分从当前 Store 的记录中删除（如果有必要可以创建新的 Span 以记录分配后产生的碎片），对于无法进行分配的 Span，从给定 `Store &s` 中删除，并放入到 `Store &diff` 中保存。


调用前：

- 当前 Store 对象保存了所有可供分配的存储空间，
- 传入的 `Store &s` 对象内保存了所有已经分配空间的 Span 对象，
- 传入的 `Store &diff` 对象内为空。

返回时：

- 当前 Store 对象保存了剩余的可分配空间，
- 传入的 `Store &s` 对象内仅保存了成功在当前 Store 对象完成分配的 Span 对象，
- 传入的 `Store &diff` 对象内保存了未能完成的分配的 Span 对象。

```
void
Store::try_realloc(Store &s, Store &diff)
{
  // 遍历指定 Store &s 对象里的所有磁盘
  for (unsigned i = 0; i < s.n_disks; i++) {
    Span *prev = 0;
    // 遍历该磁盘里的所有 Span 对象
    for (Span *sd = s.disk[i]; sd;) {
      // 遍历当前 Store 对象里的 Span 对象，查找与指定 Span 对象在空间上是否有重叠
      // ==================== BEGIN ====================
      for (unsigned j = 0; j < n_disks; j++)
        // 遍历该磁盘里的所有 Span 对象
        for (Span *d = disk[j]; d; d = d->link.next)
          // 如果发现两个 Span 对象的文件名相同
          if (!strcmp(sd->pathname, d->pathname)) {
            // 如果 Store &s 里的 Span 所对应的空间被完全包含在当前 Store 的某个 Span 里，
            // 就在当前 Store 的这个 Span 上记录该分配，从 Span 上删除掉已经分配的空间
            if (sd->offset >= d->offset && (sd->end() <= d->end())) {
              if (!sd->file_pathname || (sd->end() == d->end())) {
                // 如果两个 Span 的结尾重叠，或者 Store &s 里的 Span 的底层不是块设备
                d->blocks -= sd->blocks;
                // 然后跳转到 Lfound
                goto Lfound;
              } else if (sd->offset == d->offset) {
                // 如果两个 Span 的起点重叠
                d->blocks -= sd->blocks;
                d->offset += sd->blocks;
                // 然后跳转到 Lfound
                goto Lfound;
              } else {
                // 如果两个 Span 的起点和结尾都不重叠
                // 那么从 Span 上删除掉已经分配的空间后，原来的 Span 被分成了两部分。
                // 这里创建一个新的 Span 对象插入到 Span 链表里
                Span *x = new Span(*d);
                // d will be the first vol
                d->blocks = sd->offset - d->offset;
                d->link.next = x;
                // x will be the last vol
                x->offset = sd->offset + sd->blocks;
                x->blocks -= x->offset - d->offset;
                // 然后跳转到 Lfound
                goto Lfound;
              }
            }
          }
      // ===================== END =====================
      {
        // 如果遍历完当前 Store 内的所有 Span 对象后，仍未找到有重叠关系的 Span 对象，
        // 则将该 Span 对象从 Store &s 中删除（判断 prev 的状态以保持链表的连续性）
        if (!prev)
          s.disk[i] = s.disk[i]->link.next;
        else
          prev->link.next = sd->link.next;
        // 然后将该 Span 对象保存到 Store &diff 对象中
        diff.extend(i + 1);
        sd->link.next = diff.disk[i];
        diff.disk[i] = sd;
        // 将下一个 Span 对象设定为查找任务，继续在当前 Store 对象中查找
        sd = prev ? prev->link.next : s.disk[i];
        continue;
      }
    Lfound:
      // 将下一个 Span 对象设定为查找任务，继续在当前 Store 对象中查找
      ;
      prev = sd;
      sd = sd->link.next;
    }
  }
  // 遍历当前 Store::disk[] 指针数组，将指向 NULL 的元素删除，并重新计算 n_disks 的值
  normalize();
  // 遍历 Store &s 的 disk[] 指针数组，将指向 NULL 的元素删除，并重新计算 n_disks 的值  
  s.normalize();
  // 遍历 Store &diff 的 disk[] 指针数组，将指向 NULL 的元素删除，并重新计算 n_disks 的值 
  diff.normalize();
}
```

可以在 `Store &s` 中存储配置文件的信息，然后在当前 Store 中存储物理设备的信息，然后调用当前 Store 的 `try_realloc` 方法即可测试配置文件所描述的空间分配信息是否与实际的物理设备的存储空间相匹配。可以参考 `MultiCacheBase::open` 方法对 `try_realloc` 的调用。


## 参考资料

- [I_Store.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/cache/I_Store.h)
- [Store.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/cache/Store.cc)
- [MultiCache.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/MultiCache.cc)

