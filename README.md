# ZFS datastore for MariaDB

This documents the settings we use at Let's Encrypt to create ZFS backing storage for MariaDB, and the tips and best practices that led us here.

## Storage priorities

Our priorities are, in order:

1. Integrity
2. Performance
3. Durability

Our data must be correct. Integrity is our service's most important property. We mustn't change settings which might defeat ZFS's inherent high integrity.

We've had stubborn performance problems because of our database's large size, schema, and access patterns. We ran out of improvements to those properties that we could make in the short term. This made it a high priority to wring as much performance out of our database servers as we safely could.

Our primary database server rapidly replicates to two others, including two locations, and is backed up daily. The most business- and compliance-critical data is also logged separately, outside of our database stack. As long as we can maintain durability for long enough to evacuate the primary (write) role to a healthier database server, that is enough.

Accordingly, we're tuning ZFS and MariaDB for performance over durability, avoiding only the more dangerous tradeoffs.

## Preparing the drives

Many modern storage drives can present different sector sizes (LBA formats) to the host system. Only one (or none) will be their internal, best-performing sector size. This is often the largest sector size they can natively support, e.g. "4Kn."<sup>[1](#fn1),[2](#fn2),[3](#fn3),[4](#fn4)</sup> We currently use Intel NVMe drives, which have a changeable "Variable Sector Size."<sup>[5](#fn5)</sup> Intel's online documentation and specifications don't list the sector size options for the P4610 model we use, but scanning it showed us two possible values: 0 (512B) or 1 (4KB). flashbench<sup>[6](#fn6),[7](#fn7)</sup> results strongly suggest that the internal sector size is 8KB.

### Implementation

We use the Intel Memory & Storage Tool<sup>[8](#fn8)</sup> to set the Variable Sector Size to 4,096, the best-performing of the available options.

**WARNING:** This erases all data.
```
for driveIndex in {0..23}; do
    sudo intelmas start             \
        -intelssd ${driveIndex}     \
        -nvmeformat                 \
            LBAFormat=1             \
            SecureEraseSetting=0    \
            ProtectionInformation=0 \
            MetadataSettings=0

    sudo intelmas show              \
        -intelssd ${driveIndex}     \
        -display SectorSize
done
```
<sup>[5](#fn5),[9](#fn9)</sup>

## ZFS kernel module settings

* Almost all of the I/O to our datasets will be done by the InnoDB database engine, which has its own prefetching logic. Since ZFS's prefetching would be redundant and less well optimized, we disable it: `zfs_prefetch_disable=1`.<sup>[1](#fn1),[10](#fn10)</sup>

## Building the vdevs, pool & datasets

### Basic concepts, from the bottom up<sup>[11](#fn11)</sup>

* ZFS acts as both a volume manager and a filesystem.

* A "vdev" can be a single drive or a RAID-style set of drives.

* A "pool" is made of 1+ vdev(s). We use multiple vdevs to build a RAID-1+0-style pool.

* A "dataset" is built on a pool. It's usually used as a filesystem. You can create a hierarchy of child datasets that inherit properties from their parents.

### Vdevs & pool

* We match our drives' best-performing 8KB sector size: `ashift=13`.<sup>[1](#fn1),[2](#fn2),[3](#fn3),[4](#fn4)</sup>

* We use `/dev/disk/by-id/` paths to identify drives, in case they're swapped around to different drive bays or the OS' device naming schema changes.<sup>[3](#fn3)</sup>

* We use RAID-1+0, in order to achieve the best possible performance without being vulnerable to a single-drive failure.<sup>[3](#fn3),[10](#fn10),[12](#fn12),[13](#fn13),[14](#fn14)</sup>

* We balance vdevs across controllers, buses, and backplane segments, in order to improve throughput and fault tolerance.<sup>[3](#fn3)</sup>

* We store data in datasets, not directly in pools, in order to allow easier management of properties, quotas, and snapshots.<sup>[3](#fn3)</sup>

#### Implementation

```
sudo zpool create                                                                       \
    -o ashift=13                                                                        \
    db05                                                                                \
    mirror                                                                              \
        /dev/disk/by-id/nvme-P4610_Drive01 \
        /dev/disk/by-id/nvme-P4610_Drive02 \
    mirror                                                                              \
        /dev/disk/by-id/nvme-P4610_Drive03 \
        /dev/disk/by-id/nvme-P4610_Drive04 \

[etc.]

    mirror                                                                              \
        /dev/disk/by-id/nvme-P4610_Drive21 \
        /dev/disk/by-id/nvme-P4610_Drive22 \
    spare                                                                               \
        /dev/disk/by-id/nvme-P4610_Drive23 \
        /dev/disk/by-id/nvme-P4610_Drive24
```

### Parent dataset

* These properties are inherited by child datasets, unless overridden.

* We've no need to incur the overhead of tracking when files were last accessed: `atime=off`.<sup>[1](#fn1),[15](#fn15)</sup>

* We want to automatically activate hot spare drives if another drive fails: `autoreplace=on`.<sup>[3](#fn3)</sup>

* We use LZ4 compression, which is extremely efficient and may even improve performance by reducing I/O to the drives: `compression=lz4`.<sup>[1](#fn1),[2](#fn2),[3](#fn3),[4](#fn4),[11](#fn11),[13](#fn13),[14](#fn14)</sup>

* Just like with prefetching, InnoDB has its own caching logic, so ZFS's caching would be redundant and less well optimized. We have ZFS cache only metadata: `primarycache=metadata`.<sup>[1](#fn1),[2](#fn2),[10](#fn10),[13](#fn13)</sup>

* ZFS's default record size of 128KB is appropriate for medium-sequential writes, i.e. general use including database backups, which may also use this dataset. We set it explicitly - `recordsize=128k`<sup>[1](#fn1),[2](#fn2),[10](#fn10),[13](#fn13),[14](#fn14),[15](#fn15)</sup> - on this parent dataset, and override it on the InnoDB child dataset.

* We store extended attributes in inodes, instead of hidden subdirectories, to reduce I/O overhead for SELinux: `xattr=sa`.<sup>[1](#fn1),[4](#fn4),[16](#fn16),[17](#fn17)</sup> Use of this flag is further supported given that we rely on SELinux and POSIX ACLs in our systems. Without the flag, even the root user attempting to set an ACL on a folder/file on a ZFS mount will receive `Operation not permitted`. According to the zfs man page <sup>[21](#fn21)</sup>,
    > The use of system attribute based xattrs is strongly encouraged for users of SELinux or POSIX ACLs. Both of these features heavily rely of extended attributes and benefit significantly from the reduced access time.

* We also allow larger nodes, in order to accommodate this: `dnodesize=auto`.<sup>[4](#fn4)</sup> N.b. This does break our pools' compatibility with non-Linux ZFS implementations.

#### Implementation

```
sudo zfs create              \
    -o mountpoint=/datastore \
    -o atime=off             \
    -o autoreplace=on        \
    -o compression=lz4       \
    -o dnodesize=auto        \
    -o primarycache=metadata \
    -o recordsize=128k       \
    -o xattr=sa              \
    -o acltype=posixacl      \
    db01/mysql

sudo zfs get acltype
```

### InnoDB child dataset

* Although we're not using a ZIL device because all our drives are the same (fast) speed, we still hint to ZFS that throughput is more important than latency for our workload: `logbias=throughput`.<sup>[1](#fn1),[2](#fn2),[14](#fn14)</sup>

* InnoDB's default page size is 16KB. (This would be interesting to experiment with.) We know every write will be that size, and it's a multiple of the drives' sector size. So, we set the tablespace dataset's record size to match: `recordsize=16k`.<sup>[1](#fn1),[2](#fn2),[10](#fn10),[13](#fn13),[14](#fn14),[15](#fn15)</sup>

* ZFS stores an *extra* copy of all metadata by default, beyond the redundancy provided by mirroring. Because we're prioritizing performance for a write-intensive workload, we lower this level of redundancy: `redundant_metadata=most`.<sup>[2](#fn2),[13](#fn13)</sup>

#### Implementation

```
sudo zfs create                 \
    -o mountpoint=/datastore/db \
    -o logbias=throughput       \
    -o recordsize=16k           \
    -o redundant_metadata=most  \
    db01/mysql/innodb
```

## MariaDB settings

* ZFS has very efficient checksumming that's integral to its operation. So, we turn off InnoDB's checksums, which would be redundant: `innodb_checksum_algorithm=none`.<sup>[1](#fn1)</sup>

* Because ZFS writes are atomic and we've aligned page/record sizes, we disable the doublewrite buffer in order to reduce overhead: `innodb_doublewrite=0`.<sup>[1](#fn1),[2](#fn2),[10](#fn10),[14](#fn14),[15](#fn15)</sup>

* We store tables in individual files, for much easier backup, recovery, or relocation: `innodb_file_per_table=ON`.<sup>[13](#fn13)</sup>

* We reduce writes by setting the redo log's write-ahead block size to match the InnoDB dataset's record size, 16KB: `innodb_log_write_ahead_size=16384`.<sup>[1](#fn1)</sup> Some articles suggest using a larger block size for logs, but MySQL caps this value at the tablespace's record size.<sup>[1](#fn1),[18](#fn18)</sup>

* We disable AIO, which performs poorly on Linux: `innodb_use_native_aio=0`, `innodb_use_atomic_writes=0`.<sup>[2](#fn2)</sup>

## Operations

* We'll run regular scrubs (integrity checks) of zpools.<sup>[3](#fn3),[11](#fn11),[19](#fn19)</sup>

* We'll monitor zpools' health using Prometheus' node_exporter.<sup>[20](#fn20)</sup>

## References

<a name="fn1">1</a>: https://shatteredsilicon.net/blog/2020/06/05/mysql-mariadb-innodb-on-zfs/

<a name="fn2">2</a>: https://openzfs.org/wiki/Performance_tuning

<a name="fn3">3</a>: https://pthree.org/2012/12/13/zfs-administration-part-viii-zpool-best-practices-and-caveats/

<a name="fn4">4</a>: https://wiki.debian.org/ZFS#Advanced_Topics

<a name="fn5">5</a>: https://www.intel.com/content/www/us/en/support/articles/000016238/memory-and-storage/data-center-ssds.html

<a name="fn6">6</a>: https://github.com/bradfa/flashbench

<a name="fn7">7</a>: https://old.reddit.com/r/zfs/comments/aq913n/anyone_know_what_the_physical_sector_size_ashift/egjudnr/

<a name="fn8">8</a>: https://downloadcenter.intel.com/product/140108/intel-ssd-dc-p4610-series

<a name="fn9">9</a>: https://downloadmirror.intel.com/29821/eng/Intel_Memory_And_Storage_Tool_User%20Guide-Public-342245-004US.pdf

<a name="fn10">10</a>: http://assets.en.oreilly.com/1/event/21/Optimizing%20MySQL%20Performance%20with%20ZFS%20Presentation.pdf

<a name="fn11">11</a>: https://www.freebsd.org/doc/handbook/zfs-term.html

<a name="fn12">12</a>: https://old.reddit.com/r/zfs/comments/47i2wi/innodb_and_arc_for_workload_thats_50_update/d0erson/

<a name="fn13">13</a>: https://www.usenix.org/system/files/login/articles/login_winter16_09_jude.pdf

<a name="fn14">14</a>: https://old.reddit.com/r/zfs/comments/3mvv8e/does_anyone_run_mysql_or_postgresql_on_zfs/cvlbyjz/

<a name="fn15">15</a>: https://wiki.freebsd.org/ZFSTuningGuide#MySQL

<a name="fn16">16</a>: https://openzfs.org/wiki/Features#SA_based_xattrs

<a name="fn17">17</a>: https://old.reddit.com/r/zfs/comments/89xe9u/zol_xattrsa/

<a name="fn18">18</a>: https://dev.mysql.com/doc/refman/5.7/en/optimizing-innodb-logging.html

<a name="fn19">19</a>: https://www.bouncybouncy.net/blog/incremental-zpool-scrub/

<a name="fn20">20</a>: https://github.com/prometheus/node_exporter/pull/1632

<a name="fn21">21</a>: https://zfsonlinux.org/manpages/0.8.4/man8/zfs.8.html