---
title: "Minio文件存储"
description: 
date: 2025-02-12T16:09:16+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:
  - minio
categories:
---

##  Minio

在项目中，存储文件有三种选择：

* 存本地 一般是不选择 （带宽风险  服务器带宽是有限的，集群部署 图片是存在不同的服务器）
* 存云存储 比如阿里云存储oss 七牛云存储  收费的，针对企业用户
* 存入分布式存储系统（自己搭建） 内网搭建 也能支持 对我们售卖是有帮助的

在一般情况下，使用云存储是最佳选择，但有些时候，尤其是To B的业务，使用自己搭建的分布式存储服务更加适合。

MinIO 是一款高性能、分布式的对象存储系统. 它是一款软件产品, 可以100%的运行在标准硬件。即X86等低成本机器也能够很好的运行MinIO。

它基于Apache License 开源协议，兼容Amazon S3云存储接口。适合存储非结构化数据，如图片，音频，视频，日志等。对象文件最大可以达到5TB。 

MinIO是CNCF成员，在云原生存储部分和ceph等一起作为目前的解决方案之一。

MinIO用作云原生应用程序的主要存储，与传统对象存储相比，云原生应用程序需要更高的吞吐量和更低的延迟。而这些都是MinIO能够达成的性能指标。

### docker部署

```go
minio:
    container_name: minio
    image: bitnami/minio:2023
    ports:
      - '9009:9000'
      - '9001:9001'
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=admin123456
    volumes:
      - '${MINIO_DIR}/data:/data'
```



### Minio进行连接和文件上传

```go
type MinioClient struct {
	c *minio.Client
}

func New(endpoint, accessKey, secretKey string, useSSL bool) (*MinioClient, error) {
	// Initialize minio client object.
	minioClient, err := minio.New(endpoint, &minio.Options{
		Creds:  credentials.NewStaticV4(accessKey, secretKey, ""),
		Secure: useSSL,
	})
	return &MinioClient{c: minioClient}, err
}

func (c *MinioClient) Upload(
	ctx context.Context,
	bucket string,
	fileName string,
	contentType string,
	data []byte) (minio.UploadInfo, error) {

	object, err := c.c.PutObject(
		ctx,
		bucket,
		fileName,
		bytes.NewBuffer(data),
		int64(len(data)),
		minio.PutObjectOptions{ContentType: contentType},
	)
	return object, err
}

func (c *MinioClient) Compose(
	ctx context.Context,
	bucket string,
	fileName string,
	contentType string,
	totalChunk int) error {
	dstOpts := minio.CopyDestOptions{
		Bucket: bucket,
		Object: fileName,
	}
	var srcs []minio.CopySrcOptions
	for i := 1; i <= totalChunk; i++ {
		formatInt := strconv.FormatInt(int64(i), 10)
		src := minio.CopySrcOptions{
			Bucket: bucket,
			Object: fileName + "_" + formatInt,
		}
		srcs = append(srcs, src)
	}

	_, err := c.c.ComposeObject(context.Background(), dstOpts, srcs...)
	return err
}

```



```go
func (t *HandlerTask) uploadFiles(c *gin.Context) {
	result := &common.Result{}
	req := model.UploadFileReq{}
	c.ShouldBind(&req)
	multipartForm, err := c.MultipartForm()
	if err != nil {
		zap.L().Error("c.MultipartForm() err", zap.Error(err))
		return
	}
	file := multipartForm.File
	key := ""
	minioClient, err := mio.New(
		"localhost:9009",
		"0RnUy2AUBZrpsI3U",
		"9lHE5HpEUwHni1vxaFnf9BosILz79nd3",
		false)
	if err != nil {
		c.JSON(http.StatusOK, result.Fail(-999, err.Error()))
		return
	}
	if req.TotalChunks == 1 {
		header := file["file"][0]
		open, err := header.Open()
		defer open.Close()
		buf := make([]byte, req.TotalSize)
		open.Read(buf)

		info, err := minioClient.Upload(
			context.Background(),
			"test",
			req.Filename,
			header.Header.Get("Content-Type"),
			buf,
		)
		if err != nil {
			c.JSON(http.StatusOK, result.Fail(-999, err.Error()))
		}
		key = info.Bucket + "/" + req.Filename
	}
	if req.TotalChunks > 1 {
		buf := make([]byte, req.CurrentChunkSize)
		//分片上传 合起来即可
		header := file["file"][0]
		open, _ := header.Open()
		defer open.Close()
		open.Read(buf)
		formatInt := strconv.FormatInt(int64(req.ChunkNumber), 10)
		info, err := minioClient.Upload(
			context.Background(),
			"test",
			req.Filename+"_"+formatInt,
			header.Header.Get("Content-Type"),
			buf,
		)
		if err != nil {
			c.JSON(http.StatusOK, result.Fail(-999, err.Error()))
		}
		key = info.Bucket + "/" + req.Filename
		if req.TotalChunks == req.ChunkNumber {
			err := minioClient.Compose(
				context.Background(),
				"test",
				req.Filename,
				header.Header.Get("Content-Type"),
				req.TotalChunks,
			)
			if err != nil {
				c.JSON(http.StatusOK, result.Fail(-999, err.Error()))
			}
		}
	}
	//调用服务 存入file表
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()
	fileUrl := "http://localhost:9009/" + key
	msg := &task.TaskFileReqMessage{
		TaskCode:         req.TaskCode,
		ProjectCode:      req.ProjectCode,
		OrganizationCode: c.GetString("organizationCode"),
		PathName:         key,
		FileName:         req.Filename,
		Size:             int64(req.TotalSize),
		Extension:        path.Ext(key),
		FileUrl:          fileUrl,
		FileType:         file["file"][0].Header.Get("Content-Type"),
		MemberId:         c.GetInt64("memberId"),
	}
	if req.TotalChunks == req.ChunkNumber {
		_, err = TaskServiceClient.SaveTaskFile(ctx, msg)
		if err != nil {
			code, msg := errs.ParseGrpcError(err)
			c.JSON(http.StatusOK, result.Fail(code, msg))
		}
	}
	c.JSON(http.StatusOK, result.Success(gin.H{
		"file":        key,
		"hash":        "",
		"key":         key,
		"url":         "http://localhost:9009/" + key,
		"projectName": req.ProjectName,
	}))
	return
}
```

