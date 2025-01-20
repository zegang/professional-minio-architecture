## 3.2 存储抽象接口

抽象接口`StorageAPI`定义于`minio/cmd/storage-interface.go`文件中，其定义了：
- 盘的相关操作；
- 卷的相关操作；
- 元信息的相关操作；
- 文件的相关操作；
```
// StorageAPI interface.
type StorageAPI interface {
	// Stringified version of disk.
	String() string

	// Storage operations.

	// Returns true if disk is online and its valid i.e valid format.json.
	// This has nothing to do with if the drive is hung or not responding.
	// For that individual storage API calls will fail properly. The purpose
	// of this function is to know if the "drive" has "format.json" or not
	// if it has a "format.json" then is it correct "format.json" or not.
	IsOnline() bool

	// Returns the last time this disk (re)-connected
	LastConn() time.Time

	// Indicates if disk is local or not.
	IsLocal() bool

	// Returns hostname if disk is remote.
	Hostname() string

	// Returns the entire endpoint.
	Endpoint() Endpoint

	// Close the disk, mark it purposefully closed, only implemented for remote disks.
	Close() error

	// Returns the unique 'uuid' of this disk.
	GetDiskID() (string, error)

	// Set a unique 'uuid' for this disk, only used when
	// disk is replaced and formatted.
	SetDiskID(id string)

	// Returns healing information for a newly replaced disk,
	// returns 'nil' once healing is complete or if the disk
	// has never been replaced.
	Healing() *healingTracker
	DiskInfo(ctx context.Context, opts DiskInfoOptions) (info DiskInfo, err error)
	NSScanner(ctx context.Context, cache dataUsageCache, updates chan<- dataUsageEntry, scanMode madmin.HealScanMode, shouldSleep func() bool) (dataUsageCache, error)

	// Volume operations.
	MakeVol(ctx context.Context, volume string) (err error)
	MakeVolBulk(ctx context.Context, volumes ...string) (err error)
	ListVols(ctx context.Context) (vols []VolInfo, err error)
	StatVol(ctx context.Context, volume string) (vol VolInfo, err error)
	DeleteVol(ctx context.Context, volume string, forceDelete bool) (err error)

	// WalkDir will walk a directory on disk and return a metacache stream on wr.
	WalkDir(ctx context.Context, opts WalkDirOptions, wr io.Writer) error

	// Metadata operations
	DeleteVersion(ctx context.Context, volume, path string, fi FileInfo, forceDelMarker bool, opts DeleteOptions) error
	DeleteVersions(ctx context.Context, volume string, versions []FileInfoVersions, opts DeleteOptions) []error
	WriteMetadata(ctx context.Context, origvolume, volume, path string, fi FileInfo) error
	UpdateMetadata(ctx context.Context, volume, path string, fi FileInfo, opts UpdateMetadataOpts) error
	ReadVersion(ctx context.Context, origvolume, volume, path, versionID string, opts ReadOptions) (FileInfo, error)
	ReadXL(ctx context.Context, volume, path string, readData bool) (RawFileInfo, error)
	RenameData(ctx context.Context, srcVolume, srcPath string, fi FileInfo, dstVolume, dstPath string, opts RenameOptions) (uint64, error)

	// File operations.
	ListDir(ctx context.Context, origvolume, volume, dirPath string, count int) ([]string, error)
	ReadFile(ctx context.Context, volume string, path string, offset int64, buf []byte, verifier *BitrotVerifier) (n int64, err error)
	AppendFile(ctx context.Context, volume string, path string, buf []byte) (err error)
	CreateFile(ctx context.Context, origvolume, olume, path string, size int64, reader io.Reader) error
	ReadFileStream(ctx context.Context, volume, path string, offset, length int64) (io.ReadCloser, error)
	RenameFile(ctx context.Context, srcVolume, srcPath, dstVolume, dstPath string) error
	CheckParts(ctx context.Context, volume string, path string, fi FileInfo) error
	Delete(ctx context.Context, volume string, path string, opts DeleteOptions) (err error)
	VerifyFile(ctx context.Context, volume, path string, fi FileInfo) error
	StatInfoFile(ctx context.Context, volume, path string, glob bool) (stat []StatInfo, err error)
	ReadMultiple(ctx context.Context, req ReadMultipleReq, resp chan<- ReadMultipleResp) error
	CleanAbandonedData(ctx context.Context, volume string, path string) error

	// Write all data, syncs the data to disk.
	// Should be used for smaller payloads.
	WriteAll(ctx context.Context, volume string, path string, b []byte) (err error)

	// Read all.
	ReadAll(ctx context.Context, volume string, path string) (buf []byte, err error)
	GetDiskLoc() (poolIdx, setIdx, diskIdx int) // Retrieve location indexes.
	SetDiskLoc(poolIdx, setIdx, diskIdx int)    // Set location indexes.
	SetFormatData(b []byte)                     // Set formatData cached value
}
```
MinIO中主要有两种StorageAPI的实现：

1. **FS Storage (Filesystem Storage)**：这是一个基于文件系统的存储API，用于单个磁盘或单个挂载点的场景。在这种模式下，MinIO将所有对象存储在一个文件系统中，每个对象作为一个单独的文件。

2. **Erasure Code (XL) Storage**：这是一个基于Erasure Coding的存储API，用于多磁盘（至少4个）的场景。在这种模式下，MinIO将每个对象分割成多个数据和校验块，然后将这些块分布在所有磁盘上。这种方式提供了数据冗余和容错能力，即使在多个磁盘故障的情况下也能保护数据。

这两种StorageAPI的实现使MinIO能够适应各种不同的部署和使用场景，从单个磁盘的简单设置到大规模的多磁盘集群。

## 3.2.1 FS Storage
MinIO的FS Storage模式非常简单，只需要一个磁盘或挂载点。以下是如何使用FS Storage模式启动MinIO服务器的示例：

```bash
minio server /data
```

在这个例子中，`/data`是你想要用作存储的目录。MinIO将在这个目录中创建一个新的文件夹，用于存储所有的对象。

你可以通过访问 `http://localhost:9000` 来访问你的MinIO服务器（默认情况下，MinIO会监听9000端口）。

请注意，尽管FS Storage模式非常简单易用，但它没有提供数据冗余或容错能力。如果你的磁盘发生故障，你可能会丢失数据。因此，对于需要高可用性和数据冗余的生产环境，建议使用Erasure Code (XL) Storage模式。

## 3.2.2 XL Storage
可通过如下方式启动XL Storage：
```
minio server /data/disk1 /data/disk2 /data/disk3 /data/disk4
```

在这个例子中，MinIO将在指定的四个驱动器（/data/disk1 到 /data/disk4）之间分片数据。如果有任何一个驱动器失败，数据仍然可以从剩余的驱动器中恢复。

请记住，虽然XL存储可以提供增强的数据持久性，但由于额外的奇偶校验块，它也需要更多的存储空间。存储空间和数据持久性之间的具体权衡取决于驱动器的数量和特定的纠删码配置。

XL Storage的代码位于文件`minio/cmd/xl-storage.go`中。
```
// xlStorage - implements StorageAPI interface.
type xlStorage struct {
	// Indicate of NSScanner is in progress in this disk
	scanning int32

	drivePath string
	endpoint  Endpoint

	globalSync bool
	oDirect    bool // indicates if this disk supports ODirect

	diskID string

	// Indexes, will be -1 until assigned a set.
	poolIndex, setIndex, diskIndex int

	formatFileInfo  os.FileInfo
	formatFile      string
	formatLegacy    bool
	formatLastCheck time.Time

	diskInfoCache *cachevalue.Cache[DiskInfo]
	sync.RWMutex
	formatData []byte

	nrRequests   uint64
	major, minor uint32

	immediatePurge chan string

	// mutex to prevent concurrent read operations overloading walks.
	rotational bool
	walkMu     *sync.Mutex
	walkReadMu *sync.Mutex
}
```
### XL Storage对象的创建
`newXLStorage`函数会创建新的XL Storage对象，同时还进行了：
- 升级老版本`format.json`到新版。
- 创建所有需要的bucket文件夹。
```
// Initialize a new storage disk.
func newXLStorage(ep Endpoint, cleanUp bool) (s *xlStorage, err error) {
	immediatePurgeQueue := 100000
	if globalIsTesting || globalIsCICD {
		immediatePurgeQueue = 1
	}
	s = &xlStorage{
		drivePath:      ep.Path,
		endpoint:       ep,
		globalSync:     globalFSOSync,
		diskInfoCache:  cachevalue.New[DiskInfo](),
		poolIndex:      -1,
		setIndex:       -1,
		diskIndex:      -1,
		immediatePurge: make(chan string, immediatePurgeQueue),
	}
...
	if cleanUp {
		bgFormatErasureCleanupTmp(s.drivePath) // cleanup any old data.
	}

	formatData, formatFi, err := formatErasureMigrate(s.drivePath)
	if err != nil && !errors.Is(err, os.ErrNotExist) {
		if os.IsPermission(err) {
			return s, errDiskAccessDenied
		} else if isSysErrIO(err) {
			return s, errFaultyDisk
		}
		return s, err
	}
	s.formatData = formatData
	s.formatFileInfo = formatFi
	s.formatFile = pathJoin(s.drivePath, minioMetaBucket, formatConfigFile)

	// Create all necessary bucket folders if possible.
	if err = makeFormatErasureMetaVolumes(s); err != nil {
		return s, err
	}
```
创建所有需要的bucket文件夹。
```
// Make Erasure backend meta volumes.
func makeFormatErasureMetaVolumes(disk StorageAPI) error {
	if disk == nil {
		return errDiskNotFound
	}
	volumes := []string{
		minioMetaTmpDeletedBucket, // creates .minio.sys/tmp as well as .minio.sys/tmp/.trash
		minioMetaMultipartBucket,  // creates .minio.sys/multipart
		dataUsageBucket,           // creates .minio.sys/buckets
		minioConfigBucket,         // creates .minio.sys/config
	}
	// Attempt to create MinIO internal buckets.
	return disk.MakeVolBulk(context.TODO(), volumes...)
}
```
上面路径常量定义于文件`minio/cmd/data-usage.go`，
```
const (
	dataUsageRoot   = SlashSeparator
	dataUsageBucket = minioMetaBucket + SlashSeparator + bucketMetaPrefix

	dataUsageObjName       = ".usage.json"
	dataUsageObjNamePath   = bucketMetaPrefix + SlashSeparator + dataUsageObjName
	dataUsageBloomName     = ".bloomcycle.bin"
	dataUsageBloomNamePath = bucketMetaPrefix + SlashSeparator + dataUsageBloomName

	backgroundHealInfoPath = bucketMetaPrefix + SlashSeparator + ".background-heal.json"

	dataUsageCacheName = ".usage-cache.bin"
)
```
`minio/cmd/object-api-utils.go`，
```
const (
	// MinIO meta bucket.
	minioMetaBucket = ".minio.sys"
	// Multipart meta prefix.
	mpartMetaPrefix = "multipart"
	// MinIO Multipart meta prefix.
	minioMetaMultipartBucket = minioMetaBucket + SlashSeparator + mpartMetaPrefix
	// MinIO tmp meta prefix.
	minioMetaTmpBucket = minioMetaBucket + "/tmp"
	// MinIO tmp meta prefix for deleted objects.
	minioMetaTmpDeletedBucket = minioMetaTmpBucket + "/.trash"

	// DNS separator (period), used for bucket name validation.
	dnsDelimiter = "."
	// On compressed files bigger than this;
	compReadAheadSize = 100 << 20
	// Read this many buffers ahead.
	compReadAheadBuffers = 5
	// Size of each buffer.
	compReadAheadBufSize = 1 << 20
	// Pad Encrypted+Compressed files to a multiple of this.
	compPadEncrypted = 256
	// Disable compressed file indices below this size
	compMinIndexSize = 8 << 20
)
```
和`minio/cmd/format-meta.go`中。
```
// Format related consts
const (
	// Format config file carries backend format specific details.
	formatConfigFile = "format.json"
)

const (
	// Version of the formatMetaV1
	formatMetaVersionV1 = "1"
)

// format.json currently has the format:
// {
//   "version": "1",
//   "format": "XXXXX",
//   "XXXXX": {
//
//   }
// }
// Here "XXXXX" depends on the backend, currently we have "fs" and "xl" implementations.
// formatMetaV1 should be inherited by backend format structs. Please look at format-fs.go
// and format-xl.go for details.

// Ideally we will never have a situation where we will have to change the
// fields of this struct and deal with related migration.
type formatMetaV1 struct {
	// Version of the format config.
	Version string `json:"version"`
	// Format indicates the backend format type, supports two values 'xl' and 'fs'.
	Format string `json:"format"`
	// ID is the identifier for the minio deployment
	ID string `json:"id"`
}
```

### 卷操作
现在来看看卷的初始化，接口是`MakeVol(ctx context.Context, volume string) (err error)`.XLStorage的实现是在驱动器根目录下创建给定名字的子目录。
```
func (s *xlStorage) MakeVolBulk(ctx context.Context, volumes ...string) error {
	for _, volume := range volumes {
		err := s.MakeVol(ctx, volume)
		if err != nil && !errors.Is(err, errVolumeExists) {
			return err
		}
		diskHealthCheckOK(ctx, err)
	}
	return nil
}

// Make a volume entry.
func (s *xlStorage) MakeVol(ctx context.Context, volume string) error {
	if !isValidVolname(volume) {
		return errInvalidArgument
	}

	volumeDir, err := s.getVolDir(volume)
	if err != nil {
		return err
	}

	if err = Access(volumeDir); err != nil {
		// Volume does not exist we proceed to create.
		if osIsNotExist(err) {
			// Make a volume entry, with mode 0777 mkdir honors system umask.
			err = mkdirAll(volumeDir, 0o777, s.drivePath)
		}
		if osIsPermission(err) {
			return errDiskAccessDenied
		} else if isSysErrIO(err) {
			return errFaultyDisk
		}
		return err
	}

	// Stat succeeds we return errVolumeExists.
	return errVolumeExists
}
```
再来看看卷的查看，接口`ListVols(ctx context.Context)`。XLStorage通过驱动器根路径遍历其下的所有子目录而得到所有的卷。
```
// ListVols - list volumes.
func (s *xlStorage) ListVols(ctx context.Context) (volsInfo []VolInfo, err error) {
	return listVols(ctx, s.drivePath)
}

// List all the volumes from drivePath.
func listVols(ctx context.Context, dirPath string) ([]VolInfo, error) {
	if err := checkPathLength(dirPath); err != nil {
		return nil, err
	}
	entries, err := readDir(dirPath)
	if err != nil {
		if errors.Is(err, errFileAccessDenied) {
			return nil, errDiskAccessDenied
		} else if errors.Is(err, errFileNotFound) {
			return nil, errDiskNotFound
		}
		return nil, err
	}
	volsInfo := make([]VolInfo, 0, len(entries))
	for _, entry := range entries {
		if !HasSuffix(entry, SlashSeparator) || !isValidVolname(pathutil.Clean(entry)) {
			// Skip if entry is neither a directory not a valid volume name.
			continue
		}
		volsInfo = append(volsInfo, VolInfo{
			Name: pathutil.Clean(entry),
		})
	}
	return volsInfo, nil
}
```
### 文件操作
我们再来看看XL Storage针对Storage API的文件操作的实现。首先看看文件的读取:
```
ReadFile(ctx context.Context, volume string, path string, offset int64, buf []byte, verifier *BitrotVerifier) (n int64, err error)
```
XL Storage实现的文件读取（代码位于文件minio/cmd/xl-storage.go)：
1. 获取卷（volume）的文件夹路径；
2. 拼接文件路径；
3. 以只读方式打开文件；
4. 获取文件属性；
5. 判断是否是常规文件，不然seek操作是未定义的行为；
6. 如果没有指定位衰减（bitrot）验证器，直接进行文件读取：`file.ReadAt(buffer, offset)`，返回读取的长度和可能的错误;
7. 否则，将读取文件内容并通过指定的算法（哈希算法）进行计算，并将结果与参数传入的`verifier.sum`比对。相同则数据正确，不同则返回错误`errFileCorrupt`。
```
// ReadFile reads exactly len(buf) bytes into buf. It returns the
// number of bytes copied. The error is EOF only if no bytes were
// read. On return, n == len(buf) if and only if err == nil. n == 0
// for io.EOF.
//
// If an EOF happens after reading some but not all the bytes,
// ReadFile returns ErrUnexpectedEOF.
//
// If the BitrotVerifier is not nil or not verified ReadFile
// tries to verify whether the disk has bitrot.
//
// Additionally ReadFile also starts reading from an offset. ReadFile
// semantics are same as io.ReadFull.
func (s *xlStorage) ReadFile(ctx context.Context, volume string, path string, offset int64, buffer []byte, verifier *BitrotVerifier) (int64, error) {
	if offset < 0 {
		return 0, errInvalidArgument
	}

	volumeDir, err := s.getVolDir(volume)
	if err != nil {
		return 0, err
	}

	var n int

	if !skipAccessChecks(volume) {
		// Stat a volume entry.
		if err = Access(volumeDir); err != nil {
			return 0, convertAccessError(err, errFileAccessDenied)
		}
	}

	// Validate effective path length before reading.
	filePath := pathJoin(volumeDir, path)
	if err = checkPathLength(filePath); err != nil {
		return 0, err
	}

	// Open the file for reading.
	file, err := OpenFile(filePath, readMode, 0o666)
	if err != nil {
		switch {
		case osIsNotExist(err):
			return 0, errFileNotFound
		case osIsPermission(err):
			return 0, errFileAccessDenied
		case isSysErrNotDir(err):
			return 0, errFileAccessDenied
		case isSysErrIO(err):
			return 0, errFaultyDisk
		case isSysErrTooManyFiles(err):
			return 0, errTooManyOpenFiles
		default:
			return 0, err
		}
	}

	// Close the file descriptor.
	defer file.Close()

	st, err := file.Stat()
	if err != nil {
		return 0, err
	}

	// Verify it is a regular file, otherwise subsequent Seek is
	// undefined.
	if !st.Mode().IsRegular() {
		return 0, errIsNotRegular
	}

	if verifier == nil {
		n, err = file.ReadAt(buffer, offset)
		return int64(n), err
	}

	h := verifier.algorithm.New()
	if _, err = io.Copy(h, io.LimitReader(file, offset)); err != nil {
		return 0, err
	}

	if n, err = io.ReadFull(file, buffer); err != nil {
		return int64(n), err
	}

	if _, err = h.Write(buffer); err != nil {
		return 0, err
	}

	if _, err = io.Copy(h, file); err != nil {
		return 0, err
	}

	if !bytes.Equal(h.Sum(nil), verifier.sum) {
		return 0, errFileCorrupt
	}

	return int64(len(buffer)), nil
}
```
上面位衰减（bitrot）验证器定义于文件`minio/cmd/bitrot.go`中。
```
// BitrotVerifier can be used to verify protected data.
type BitrotVerifier struct {
	algorithm BitrotAlgorithm
	sum       []byte
}
```
算法类型`BitrotAlgorithm`就是Go内置类型`uint`的一个别名，通过该类型定义了4个算法常量。参见文件`minio/cmd/xl-storage-format-v1.go`。这在Go中是一个很常见的写法，用于限定值的有效范围。
其后的代码还指定了默认的算法为HighwayHash256S。
```
// BitrotAlgorithm specifies a algorithm used for bitrot protection.
type BitrotAlgorithm uint

const (
	// SHA256 represents the SHA-256 hash function
	SHA256 BitrotAlgorithm = 1 + iota
	// HighwayHash256 represents the HighwayHash-256 hash function
	HighwayHash256
	// HighwayHash256S represents the Streaming HighwayHash-256 hash function
	HighwayHash256S
	// BLAKE2b512 represents the BLAKE2b-512 hash function
	BLAKE2b512
)

// DefaultBitrotAlgorithm is the default algorithm used for bitrot protection.
const (
	DefaultBitrotAlgorithm = HighwayHash256S
)
```
类型`BitrotAlgorithm`的`New`方法位于文件`minio/cmd/bitrot.go`中，其返回值是Go中代表哈希算法的`hash.Hash`接口。该方法将创建特定的哈希算法并返回。
```
import (
...
	"github.com/minio/highwayhash"
	"github.com/minio/minio/internal/hash/sha256"
	"golang.org/x/crypto/blake2b"
...
)

// magic HH-256 key as HH-256 hash of the first 100 decimals of π as utf-8 string with a zero key.
var magicHighwayHash256Key = []byte("\x4b\xe7\x34\xfa\x8e\x23\x8a\xcd\x26\x3e\x83\xe6\xbb\x96\x85\x52\x04\x0f\x93\x5d\xa3\x9f\x44\x14\x97\xe0\x9d\x13\x22\xde\x36\xa0")

var bitrotAlgorithms = map[BitrotAlgorithm]string{
	SHA256:          "sha256",
	BLAKE2b512:      "blake2b",
	HighwayHash256:  "highwayhash256",
	HighwayHash256S: "highwayhash256S",
}

// New returns a new hash.Hash calculating the given bitrot algorithm.
func (a BitrotAlgorithm) New() hash.Hash {
	switch a {
	case SHA256:
		return sha256.New()
	case BLAKE2b512:
		b2, _ := blake2b.New512(nil) // New512 never returns an error if the key is nil
		return b2
	case HighwayHash256:
		hh, _ := highwayhash.New(magicHighwayHash256Key) // New will never return error since key is 256 bit
		return hh
	case HighwayHash256S:
		hh, _ := highwayhash.New(magicHighwayHash256Key) // New will never return error since key is 256 bit
		return hh
	default:
		logger.CriticalIf(GlobalContext, errors.New("Unsupported bitrot algorithm"))
		return nil
	}
}
```