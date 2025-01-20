## 1.4 Minio的源码

### 1.4.1 Minio源码的目录结构

- buildscripts:
- cmd: Minio的命令行处理包，包含了所有的命令行处理函数。
- dockersripts: Docker相关的脚本。
- docs: 文档。
- helm: Helm相关的部分。
- internal: Minio的内部包，包含了所需的功能代码，库代码。比如：
  - arn: Amazon Resource Names (ARNs) ，资源的唯一标识；
  - auth: 安全认证相关的代码；
  - bpool: 字节池（byte pool）；
  - bucket: 桶相关的代码；
  - disk: 盘相关的代码；
  - etag: S3的ETag支持。通常每个S3对象有一个关联的ETag，值常为对象内容的MD5检验和（也可以不是，如若通过S3 Multi-part Upload的对象）。方便快速比对和判断对象是否有更改；
  - hash: 哈希函数相关的代码；
  - ioutil: IO相关的代码，需要系统相关考虑（Unix, Windows）；
  - jwt: JSON Web Token。一种紧凑、URL 安全的方式，用于在两方之间传递声明。JWT 中的声明被编码为 JSON 对象，该对象用作 JSON Web Signature (JWS) 结构的有效载荷，或作为 JSON Web Encryption (JWE) 结构的明文，使得声明可以被数字签名或用消息认证码（MAC）进行完整性保护，和/或被加密;
  - kms: Key管理子系统；
  - lock：锁机制相关的代码；
  - s3select: 针对支持S3 Select API的支持。S3 Select是Amazon S3的一项功能，旨在从对象中提取您需要的数据，这可以显著提高需要访问S3数据的应用程序的性能并降低成本。MinIO允许使用SQL表达式从JSON，CSV文件或Parquet检索数据的子集。它允许将过滤和访问对象内容的繁重工作卸载到MinIO服务器。

### 1.4.2 Minio源码编译

### 1.4.2.1 Go on Ubuntu

Ubuntu 18.04, 20.04 or 22.04 (amd64, arm64 or armhf)，安装Golang :
```
$ sudo add-apt-repository ppa:longsleep/golang-backports
$ sudo apt update
$ sudo apt install golang-go
```
检查版本:
```
$ go version
go version go1.23.4 linux/amd64
$ go env -w GOPROXY=https://proxy.golang.org,direct
$ go env GOPROXY
https://proxy.golang.org,direct
```

### 1.4.2.1 源码编译
编译命令:
```
$ make
Checking dependencies
Building minio binary to './minio'
go: downloading cloud.google.com/go v0.116.0
go: downloading cloud.google.com/go/auth v0.13.0
go: downloading cloud.google.com/go/compute/metadata v0.6.0
go: downloading cloud.google.com/go/auth/oauth2adapt v0.2.6
go: downloading github.com/googleapis/gax-go/v2 v2.14.0
go: downloading cloud.google.com/go/iam v1.2.2
go: downloading go.opentelemetry.io/contrib/detectors/gcp v1.32.0
go: downloading go.opentelemetry.io/otel v1.33.0
go: downloading go.opentelemetry.io/otel/sdk/metric v1.32.0
go: downloading go.opentelemetry.io/otel/sdk v1.33.0
go: downloading github.com/envoyproxy/go-control-plane v0.13.1
go: downloading github.com/GoogleCloudPlatform/opentelemetry-operations-go/exporter/metric v0.49.0
go: downloading go.opentelemetry.io/otel/metric v1.33.0
go: downloading go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp v0.58.0
go: downloading go.opentelemetry.io/otel/trace v1.33.0
go: downloading go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc v0.57.0
go: downloading github.com/cncf/xds/go v0.0.0-20240905190251-b4127c9b8d78
go: downloading go.opencensus.io v0.24.0
go: downloading github.com/GoogleCloudPlatform/opentelemetry-operations-go/detectors/gcp v1.25.0
go: downloading cloud.google.com/go/monitoring v1.21.2
go: downloading github.com/GoogleCloudPlatform/opentelemetry-operations-go/internal/resourcemapping v0.49.0
go: downloading github.com/felixge/httpsnoop v1.0.4
go: downloading cel.dev/expr v0.18.0
go: downloading github.com/envoyproxy/protoc-gen-validate v1.1.0
go: downloading github.com/go-logr/logr v1.4.2
go: downloading github.com/go-logr/stdr v1.2.2
go: downloading go.opentelemetry.io/auto/sdk v1.1.0
go: downloading github.com/golang/groupcache v0.0.0-20210331224755-41bb18bfe9da
go: downloading github.com/census-instrumentation/opencensus-proto v0.4.1
```
成功后获得minio可行性文件:
```
$ ./minio --version
minio version DEVELOPMENT.2025-01-19T23-13-17Z (commit-id=779ec8f0d423ee8114ef752628f4208c69c78445)
Runtime: go1.23.4 linux/amd64
License: GNU AGPLv3 - https://www.gnu.org/licenses/agpl-3.0.html
Copyright: 2015-2025 MinIO, Inc.
```