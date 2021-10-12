---
title: FastDFS Clinet Java 源码分析
date: 2021-08-01 18:54:14
updated: 2021-08-01 18:54:14
categories: FastDFS
tags: 
  - fastdfs
  - 源码分析
---

## 什么是 FastDFS

>[FastDFS](https://github.com/happyfish100/fastdfs)

根据项目的描述，FastDFS 是一个开源的高性能分布式文件系统。 它的主要功能包括：文件存储，文件同步和文件访问，以及高容量和负载平衡。<!--more-->

## FastDFS 框架设计思路

![image(4)](fastdfs-record/image(4)-2663185.png)

FastDFS 文件系统总体上划分为三个部分，Client，Tracker Server 和 Storage Server。

1. Storage Server ，存储服务器，负责存储数据。Storage Server 以组 （group）为单位组织，一个 group 可以有多台 Storage，数据互为冗余备份。
2. Tracker Server，负责管理 Storage Server。客户端在操作时统一通过 Tracker Server 来访问服务。
3. Client 选择 Tracker Server 进行操作。

客户端上传文件，Tracker Server 选择存储的 group，在 group 中选择一个 Storage。当文件存储后，会为该文件生成一个特定的文件名，后续客户端需要通过该文件名来访问文件。

## FastDFS-Client-Java

>[FastDFS-Client-Java](https://github.com/happyfish100/fastdfs-client-java)

引入依赖，中心仓库没有的话可以手动下载源码 install。

```plain
      <dependency>
          <groupId>org.csource</groupId>
          <artifactId>fastdfs-client-java</artifactId>
          <version>1.29-SNAPSHOT</version>
      </dependency>
```

ClientGlobal 加载配置，有多种方式，.conf，.properties ，Properties 对象注入均可，具体看官方链接说明（上面链接）

ClientGlobal 中的 TrackerGroup g_tracker_group; 字段用来保存配置的地址等信息。

初始化客户端，通过 StorageClient  提供的 API 操作文件。

![image(5)](fastdfs-record/image(5).png)

以文件上传为例：

客户端调用 StorageClient 提供的**upload_file**(... args)  方法。

![image(6)](fastdfs-record/image(6).png)

所有暴露出来的接口，最终都由 **do_upload_file()** 执行。

**do_upload_file(）**先保证当前 Client 有可用的 storageServer 连接，通过 **newxxxxStorageConnection()** 保证。

![image(7)](fastdfs-record/image(7).png)

后通过 storageServer 获取 Connection。

![image(8)](fastdfs-record/image(8).png)

上传成功后，返回 group 和根据内部规则生成的 remote_file_name（后续访问文件则需要根据这个信息进行访问）。

![image(9)](fastdfs-record/image(9).png)

最后根据是否开启了连接池配置，返回连接或者关闭连接。

![image(10)](fastdfs-record/image(10).png)
