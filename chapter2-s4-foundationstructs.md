## 2.4 基础数据结构

对象存储系统有很多基础的概念，需要有响应的数据结构抽象实现。本节将展示Minio对象存储中的基础数据结构。并将展示一些数据结构(`minio server /miniodata/data`)的实例状态。

### 2.4.1 对象

对象抽象API层。
```
// ObjectLayer implements primitives for object API layer.
type ObjectLayer interface {
	// Locking operations on object.
	NewNSLock(bucket string, objects ...string) RWLocker
	...
	// Bucket operations.
	MakeBucket(ctx context.Context, bucket string, opts MakeBucketOptions) error
	GetBucketInfo(ctx context.Context, bucket string, opts BucketOptions) (bucketInfo BucketInfo, err error)
	ListBuckets(ctx context.Context, opts BucketOptions) (buckets []BucketInfo, err error)
	DeleteBucket(ctx context.Context, bucket string, opts DeleteBucketOptions) error
	ListObjects(ctx context.Context, bucket, prefix, marker, delimiter string, maxKeys int) (result ListObjectsInfo, err error)
	ListObjectsV2(ctx context.Context, bucket, prefix, continuationToken, delimiter string, maxKeys int, fetchOwner bool, startAfter string) (result ListObjectsV2Info, err error)
	ListObjectVersions(ctx context.Context, bucket, prefix, marker, versionMarker, delimiter string, maxKeys int) (result ListObjectVersionsInfo, err error)
	// Walk lists all objects including versions, delete markers.
	Walk(ctx context.Context, bucket, prefix string, results chan<- itemOrErr[ObjectInfo], opts WalkOptions) error

	// Object operations.

	// GetObjectNInfo returns a GetObjectReader that satisfies the
	// ReadCloser interface. The Close method runs any cleanup
	// functions, so it must always be called after reading till EOF
	//
	// IMPORTANTLY, when implementations return err != nil, this
	// function MUST NOT return a non-nil ReadCloser.
	GetObjectNInfo(ctx context.Context, bucket, object string, rs *HTTPRangeSpec, h http.Header, opts ObjectOptions) (reader *GetObjectReader, err error)
	GetObjectInfo(ctx context.Context, bucket, object string, opts ObjectOptions) (objInfo ObjectInfo, err error)
	PutObject(ctx context.Context, bucket, object string, data *PutObjReader, opts ObjectOptions) (objInfo ObjectInfo, err error)
	CopyObject(ctx context.Context, srcBucket, srcObject, destBucket, destObject string, srcInfo ObjectInfo, srcOpts, dstOpts ObjectOptions) (objInfo ObjectInfo, err error)
	DeleteObject(ctx context.Context, bucket, object string, opts ObjectOptions) (ObjectInfo, error)
	DeleteObjects(ctx context.Context, bucket string, objects []ObjectToDelete, opts ObjectOptions) ([]DeletedObject, []error)
	TransitionObject(ctx context.Context, bucket, object string, opts ObjectOptions) error
	RestoreTransitionedObject(ctx context.Context, bucket, object string, opts ObjectOptions) error
	...
```

### 2.4.2 端点

端点(Endpoint）存储了服务点或实例的详细信息。
```
// Endpoint - any type of endpoint.
type Endpoint struct {
	*url.URL
	IsLocal bool

	PoolIdx, SetIdx, DiskIdx int
}
```
![A screenshot of endpoint instance](images/c2-s4-endpoint-instance.png)

一个储存池中的端点由`PoolEndpoints`类型表示。
```
// PoolEndpoints represent endpoints in a given pool
// along with its setCount and setDriveCount.
type PoolEndpoints struct {
	// indicates if endpoints are provided in non-ellipses style
	Legacy       bool
	SetCount     int
	DrivesPerSet int
	Endpoints    Endpoints
	CmdLine      string
	Platform     string
}

// EndpointServerPools - list of list of endpoints
type EndpointServerPools []PoolEndpoints
```
### 2.4.3 节点
集群中的节点:
```
// Node holds information about a node in this cluster
type Node struct {
	*url.URL
	Pools    []int
	IsLocal  bool
	GridHost string
}
```
当前集群只有一个当前机器节点。
![A screenshot of nodes](images/c2-s4-cluster-nodes-instance.png)

### 2.4.4 配置类型

很明显我们所使用的启动命令是：单驱动器的配置类型（ErasureSDSetupType）
```
// SetupType - enum for setup type.
type SetupType int

const (
	// UnknownSetupType - starts with unknown setup type.
	UnknownSetupType SetupType = iota

	// FSSetupType - FS setup type enum.
	FSSetupType

	// ErasureSDSetupType - Erasure single drive setup enum.
	ErasureSDSetupType

	// ErasureSetupType - Erasure setup type enum.
	ErasureSetupType

	// DistErasureSetupType - Distributed Erasure setup type enum.
	DistErasureSetupType
)
```

### 2.4.5 事件

事件（Event）是用来通知特定行为，动作或状态的发生，需要呈现时间，地点，主角已经相关的状态信息。Minio将其抽象为`Event`类型（位于文件`/minio/internal/event/event.go`中）：
```
// Event represents event notification information defined in
// http://docs.aws.amazon.com/AmazonS3/latest/dev/notification-content-structure.html.
type Event struct {
	EventVersion      string            `json:"eventVersion"`
	EventSource       string            `json:"eventSource"`
	AwsRegion         string            `json:"awsRegion"`
	EventTime         string            `json:"eventTime"`
	EventName         Name              `json:"eventName"`
	UserIdentity      Identity          `json:"userIdentity"`
	RequestParameters map[string]string `json:"requestParameters"`
	ResponseElements  map[string]string `json:"responseElements"`
	S3                Metadata          `json:"s3"`
	Source            Source            `json:"source"`
	Type              madmin.TraceType  `json:"-"`
}
```
事件中通常需要描述相关的桶和对象，分别抽象为如下的数据结构：
```
// Bucket represents bucket metadata of the event.
type Bucket struct {
	Name          string   `json:"name"`
	OwnerIdentity Identity `json:"ownerIdentity"`
	ARN           string   `json:"arn"`
}

// Object represents object metadata of the event.
type Object struct {
	Key          string            `json:"key"`
	Size         int64             `json:"size,omitempty"`
	ETag         string            `json:"eTag,omitempty"`
	ContentType  string            `json:"contentType,omitempty"`
	UserMetadata map[string]string `json:"userMetadata,omitempty"`
	VersionID    string            `json:"versionId,omitempty"`
	Sequencer    string            `json:"sequencer"`
}
```
客户端的抽象：
```
// Source represents client information who triggered the event.
type Source struct {
	Host      string `json:"host"`
	Port      string `json:"port"`
	UserAgent string `json:"userAgent"`
}
```