## 1.4 Minio的源码概览

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