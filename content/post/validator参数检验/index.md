---
title: "validator参数检验"
description: 
date: 2024-09-12T21:33:24+08:00
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

## validator库参数校验

对参数校验是web开发中必不可少的，而手动检验重复性代码太多，所以我们引入validator库来学习参数校验

```go
	//手动对请求参数进行业务规则校验
	if len(p.Username) == 0 || len(p.Password) == 0 || len(p.RePassword) == 0 || len(p.RePassword) != len(p.Password) {
		zap.L().Error("SignUp with invalid param")
		c.JSON(http.StatusOK, gin.H{
			"msg": "请求参数有误",
		})
		return
	}
```



### validator库基本实例

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type SignUpParam struct {
	Age        uint8  `json:"age" binding:"gte=1,lte=130"`
	Name       string `json:"name" binding:"required"`
	Email      string `json:"email" binding:"required,email"`
	Password   string `json:"password" binding:"required"`
	RePassword string `json:"re_password" binding:"required,eqfield=Password"`
}

func main() {
	r := gin.Default()

	r.POST("/signup", func(c *gin.Context) {
		var u SignUpParam
		if err := c.ShouldBind(&u); err != nil {
			c.JSON(http.StatusOK, gin.H{
				"msg": err.Error(),
			})
			return
		}
		// 保存入库等业务逻辑代码...

		c.JSON(http.StatusOK, "success")
	})

	_ = r.Run(":8999")
}

```

### 翻译校验错误提示信息

`validator`库本身是支持国际化的，借助相应的语言包可以实现校验错误提示信息的自动翻译。下面的示例代码演示了如何将错误提示信息翻译成中文，翻译成其他语言的方法类似。

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/locales/en"
	"github.com/go-playground/locales/zh"
	ut "github.com/go-playground/universal-translator"
	"github.com/go-playground/validator/v10"
	enTranslations "github.com/go-playground/validator/v10/translations/en"
	zhTranslations "github.com/go-playground/validator/v10/translations/zh"
)

// 定义一个全局翻译器T
var trans ut.Translator

// InitTrans 初始化翻译器
func InitTrans(locale string) (err error) {
	// 修改gin框架中的Validator引擎属性，实现自定制
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {

		zhT := zh.New() // 中文翻译器
		enT := en.New() // 英文翻译器

		// 第一个参数是备用（fallback）的语言环境
		// 后面的参数是应该支持的语言环境（支持多个）
		// uni := ut.New(zhT, zhT) 也是可以的
		uni := ut.New(enT, zhT, enT)

		// locale 通常取决于 http 请求头的 'Accept-Language'
		var ok bool
		// 也可以使用 uni.FindTranslator(...) 传入多个locale进行查找
		trans, ok = uni.GetTranslator(locale)
		if !ok {
			return fmt.Errorf("uni.GetTranslator(%s) failed", locale)
		}

		// 注册翻译器
		switch locale {
		case "en":
			err = enTranslations.RegisterDefaultTranslations(v, trans)
		case "zh":
			err = zhTranslations.RegisterDefaultTranslations(v, trans)
		default:
			err = enTranslations.RegisterDefaultTranslations(v, trans)
		}
		return
	}
	return
}

type SignUpParam struct {
	Age        uint8  `json:"age" binding:"gte=1,lte=130"`
	Name       string `json:"name" binding:"required"`
	Email      string `json:"email" binding:"required,email"`
	Password   string `json:"password" binding:"required"`
	RePassword string `json:"re_password" binding:"required,eqfield=Password"`
}

func main() {
	if err := InitTrans("zh"); err != nil {
		fmt.Printf("init trans failed, err:%v\n", err)
		return
	}

	r := gin.Default()

	r.POST("/signup", func(c *gin.Context) {
		var u SignUpParam
		if err := c.ShouldBind(&u); err != nil {
			// 获取validator.ValidationErrors类型的errors
			errs, ok := err.(validator.ValidationErrors)
			if !ok {
				// 非validator.ValidationErrors类型错误直接返回
				c.JSON(http.StatusOK, gin.H{
					"msg": err.Error(),
				})
				return
			}
			// validator.ValidationErrors类型错误则进行翻译
			c.JSON(http.StatusOK, gin.H{
				"msg":errs.Translate(trans),
			})
			return
		}
		// 保存入库等具体业务逻辑代码...

		c.JSON(http.StatusOK, "success")
	})

	_ = r.Run(":8999")
}

```

### 自定义错误提示信息的字段名

但是上述代码的翻译错误信息中的字段为结构体字段不为tag的json字段，我们通过一个函数来进行更改

只需要在初始化翻译器的时候像下面一样添加一个获取`json` tag的自定义方法即可。

```
// InitTrans 初始化翻译器
func InitTrans(locale string) (err error) {
	// 修改gin框架中的Validator引擎属性，实现自定制
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {

		// 注册一个获取json tag的自定义方法
		v.RegisterTagNameFunc(func(fld reflect.StructField) string {
			name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
			if name == "-" {
				return ""
			}
			return name
		})

		zhT := zh.New() // 中文翻译器
		enT := en.New() // 英文翻译器

		// 第一个参数是备用（fallback）的语言环境
		// 后面的参数是应该支持的语言环境（支持多个）
		// uni := ut.New(zhT, zhT) 也是可以的
		uni := ut.New(enT, zhT, enT)

		// ... liwenzhou.com ...
}

```

```
{"msg":{"SignUpParam.email":"email必须是一个有效的邮箱","SignUpParam.password":"password为必填字段","SignUpParam.re_password":"re_password为必填字段"}}

```

在报错效果中，我们也不需要结构体字段，我们这时需要一个函数来去除错误信息的结构体名称字段

这里参考https://github.com/go-playground/validator/issues/633#issuecomment-654382345提供的方法，定义一个去掉结构体名称前缀的自定义方法：

```
func removeTopStruct(fields map[string]string) map[string]string {
	res := map[string]string{}
	for field, err := range fields {
		res[field[strings.Index(field, ".")+1:]] = err
	}
	return res
}

```

我们在代码中使用上述函数将翻译后的`errors`做一下处理即可：

```go
if err := c.ShouldBind(&u); err != nil {
	// 获取validator.ValidationErrors类型的errors
	errs, ok := err.(validator.ValidationErrors)
	if !ok {
		// 非validator.ValidationErrors类型错误直接返回
		c.JSON(http.StatusOK, gin.H{
			"msg": err.Error(),
		})
		return
	}
	// validator.ValidationErrors类型错误则进行翻译
	// 并使用removeTopStruct函数去除字段名中的结构体名称标识
	c.JSON(http.StatusOK, gin.H{
		"msg": removeTopStruct(errs.Translate(trans)),
	})
	return
}

```

最终效果

```
{"msg":{"email":"email必须是一个有效的邮箱","password":"password为必填字段","re_password":"re_password为必填字段"}}

```

### 自定义结构体校验方法

上面的校验还是有点小问题，就是当涉及到一些复杂的校验规则，比如`re_password`字段需要与`password`字段的值相等这样的校验规则，我们的自定义错误提示字段名称方法就不能很好解决错误提示信息中的其他字段名称了。

```
{"msg":{"email":"email必须是一个有效的邮箱","re_password":"re_password必须等于Password"}}

```

我们为`SignUpParam`自定义一个校验方法如下：

```
// SignUpParamStructLevelValidation 自定义SignUpParam结构体校验函数
func SignUpParamStructLevelValidation(sl validator.StructLevel) {
	su := sl.Current().Interface().(SignUpParam)

	if su.Password != su.RePassword {
		// 输出错误提示信息，最后一个参数就是传递的param
		sl.ReportError(su.RePassword, "re_password", "RePassword", "eqfield", "password")
	}
}

```

注册一下

```
func InitTrans(locale string) (err error) {
	// 修改gin框架中的Validator引擎属性，实现自定制
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {

    
		// 为SignUpParam注册自定义校验方法
		v.RegisterStructValidation(SignUpParamStructLevelValidation, SignUpParam{})

		zhT := zh.New() // 中文翻译器
		enT := en.New() // 英文翻译器
		
}

```

