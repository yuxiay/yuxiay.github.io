---
title: "gin-swagger实战"
description: 
date: 2024-10-13T22:30:21+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:   
     - go
categories:
      - go
---

## gin-swagger实战

步骤：

1. 按照swagger要求给接口代码添加声明式注释
2. 使用swag工具扫描代码自动生成API接口文档
3. 使用gin-swagger渲染在线接口文档页面

### 步骤一

在程序入口main函数上以注释的方式写下项目相关介绍信息。

```
package main

// @title 这里写标题
// @version 1.0
// @description 这里写描述信息
// @termsOfService http://swagger.io/terms/

// @contact.name 这里写联系人信息
// @contact.url http://www.swagger.io/support
// @contact.email support@swagger.io

// @license.name Apache 2.0
// @license.url http://www.apache.org/licenses/LICENSE-2.0.html

// @host 这里写接口服务的host
// @BasePath 这里写base path
func main() {
	r := gin.New()

	// liwenzhou.com ...

	r.Run()
}

```

在你代码中处理请求的接口函数（通常位于controller层）按如下方式写上注释：

```go
// GetPostListHandler2 升级版帖子列表接口
// @Summary 升级版帖子列表接口
// @Description 可按社区按时间或分数排序查询帖子列表接口
// @Tags 帖子相关接口
// @Accept application/json
// @Produce application/json
// @Param Authorization header string false "Bearer 用户令牌"
// @Param object query models.ParamPostList false "查询参数"
// @Security ApiKeyAuth
// @Success 200 {object} _ResponsePostList
// @Router /posts2 [get]
func GetPostListHandler2(c *gin.Context) {
	// GET请求参数(query string)：/api/v1/posts2?page=1&size=10&order=time
	// 初始化结构体时指定初始参数
	p := &models.ParamPostList{
		Page:  1,
		Size:  10,
		Order: models.OrderTime,
	}

	if err := c.ShouldBindQuery(p); err != nil {
		zap.L().Error("GetPostListHandler2 with invalid params", zap.Error(err))
		ResponseError(c, CodeInvalidParam)
		return
	}
	data, err := logic.GetPostListNew(p)
	// 获取数据
	if err != nil {
		zap.L().Error("logic.GetPostList() failed", zap.Error(err))
		ResponseError(c, CodeServerBusy)
		return
	}
	ResponseSuccess(c, data)
	// 返回响应
}

```



在你代码中处理请求的接口函数（通常位于controller层）按如下方式写上注释：

```go
// GetPostListHandler2 升级版帖子列表接口
// @Summary 升级版帖子列表接口
// @Description 可按社区按时间或分数排序查询帖子列表接口
// @Tags 帖子相关接口
// @Accept application/json
// @Produce application/json
// @Param Authorization header string false "Bearer 用户令牌"
// @Param object query models.ParamPostList false "查询参数"
// @Security ApiKeyAuth
// @Success 200 {object} _ResponsePostList
// @Router /posts2 [get]
func GetPostListHandler2(c *gin.Context) {
	// GET请求参数(query string)：/api/v1/posts2?page=1&size=10&order=time
	// 初始化结构体时指定初始参数
	p := &models.ParamPostList{
		Page:  1,
		Size:  10,
		Order: models.OrderTime,
	}

	if err := c.ShouldBindQuery(p); err != nil {
		zap.L().Error("GetPostListHandler2 with invalid params", zap.Error(err))
		ResponseError(c, CodeInvalidParam)
		return
	}
	data, err := logic.GetPostListNew(p)
	// 获取数据
	if err != nil {
		zap.L().Error("logic.GetPostList() failed", zap.Error(err))
		ResponseError(c, CodeServerBusy)
		return
	}
	ResponseSuccess(c, data)
	// 返回响应
}
```

上面注释中参数类型使用了`object`，`models.ParamPostList`具体定义如下：

```go
// bluebell/models/params.go

// ParamPostList 获取帖子列表query string参数
type ParamPostList struct {
	CommunityID int64  `json:"community_id" form:"community_id"`   // 可以为空
	Page        int64  `json:"page" form:"page" example:"1"`       // 页码
	Size        int64  `json:"size" form:"size" example:"10"`      // 每页数据量
	Order       string `json:"order" form:"order" example:"score"` // 排序依据
}
```

响应数据类型也使用的`object`，我个人习惯在controller层专门定义一个`docs_models.go`文件来存储文档中使用的响应数据model。

```go
// bluebell/controller/docs_models.go

// _ResponsePostList 帖子列表接口响应数据
type _ResponsePostList struct {
	Code    ResCode                 `json:"code"`    // 业务响应状态码
	Message string                  `json:"message"` // 提示信息
	Data    []*models.ApiPostDetail `json:"data"`    // 数据
}
```

------

### **第二步：生成接口文档数据**

#### **1. 安装 `swag` 工具**

bash

复制

```bash
# 安装swag命令行工具（需提前配置GOPATH环境变量）
go install github.com/swaggo/swag/cmd/swag@latest

# 验证安装
swag -v
# 输出示例: swag version v1.8.12
```

#### **2. 生成Swagger文档**

在项目根目录执行命令：

bash

复制

```bash
swag init --parseDependency --parseInternal --parseDepth 3
```

**参数说明**：

- `--parseDependency`：解析依赖包中的注释（如自定义模型）
- `--parseInternal`：解析内部依赖
- `--parseDepth 3`：解析目录深度（确保扫描到所有控制器）

**生成结果**：

```
├── docs/
│   ├── docs.go         # 文档结构代码
│   ├── swagger.json    # OpenAPI规范文件
│   └── swagger.yaml    # YAML格式文档
```

------

### **第三步：集成gin-swagger**

#### **1. 导入依赖**

在 `main.go` 中添加：

go

复制

```go
import (
    // 其他导入...
    _ "your_project/docs"  // 替换为你的项目路径（如github.com/yourname/bluebell/docs）
    ginSwagger "github.com/swaggo/gin-swagger"
    "github.com/swaggo/gin-swagger/swaggerFiles"
)
```

#### **2. 添加Swagger路由**

在Gin路由初始化代码前添加：

go

复制

```go
// main.go的main函数中
r := gin.New()

// 添加Swagger路由
r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

// 其他路由注册...
r.Run(":8080")
```

------

### **验证文档生成**

#### **1. 启动服务**

bash

复制

```bash
go run main.go
```

#### **2. 访问Swagger UI**

浏览器打开：
`http://localhost:8080/swagger/index.html`

