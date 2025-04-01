---
title: "zap日志库"
description: 
date: 2024-09-03T20:04:06+08:00
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

```go
func Init(mode string) (err error) {
	//读取配置中的logger信息,并进行配置
	//Lumberjack Logger采用以下属性作为输入:
	//
	//Filename: 日志文件的位置
	//MaxSize：在进行切割之前，日志文件的最大大小（以MB为单位）
	//MaxBackups：保留旧文件的最大个数
	//MaxAges：保留旧文件的最大天数
	//Compress：是否压缩/归档旧文件
	writeSynced := getLogWriter(
		settings.Conf.LogConfig.Filename,
		settings.Conf.LogConfig.MaxSize,
		settings.Conf.LogConfig.MaxBackups,
		settings.Conf.LogConfig.MaxAge,
	)
    //这是一个 zapcore.Encoder 类型的参数，负责将日志级别的数据（如日志消息、键值对等）编码成字节流。
	encoder := getEncoder()
    
    //定义一下日志级别
    //Zap 支持多种日志级别，包括 Debug、Info、Warn、Error、DPanic、Panic 和 Fatal
	var l = new(zapcore.Level)
	err = l.UnmarshalText([]byte(settings.Conf.LogConfig.Level))
	if err != nil {
		fmt.Printf("UnmarshalText err:%v\n", err)
		return
	}
	var core zapcore.Core
	if mode == "dev" {
		//开发模式，日主输出到终端
		consoleEncoder := zapcore.NewConsoleEncoder(zap.NewDevelopmentEncoderConfig())
		//日志输出位置，定为俩个
		core = zapcore.NewTee(
			//日志文件
			zapcore.NewCore(encoder, writeSynced, l),
			//打印到终端
			zapcore.NewCore(consoleEncoder, zapcore.Lock(os.Stdout), zapcore.DebugLevel),
		)
	} else {
		core = zapcore.NewCore(encoder, writeSynced, l)
	}

	//创建一个新的zap日志记录器
	lg := zap.New(core, zap.AddCaller())
	//替换zap库中全局的logger
	//在主函数中使用zap.L()调用全局的logger
	zap.ReplaceGlobals(lg)
	return
}

//将其更改为josn编码模式，将日志改为已读的年月日形式
func getEncoder() zapcore.Encoder {
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
	encoderConfig.TimeKey = "time"
	encoderConfig.EncodeLevel = zapcore.CapitalLevelEncoder
	encoderConfig.EncodeDuration = zapcore.SecondsDurationEncoder
	encoderConfig.EncodeCaller = zapcore.ShortCallerEncoder
	return zapcore.NewJSONEncoder(encoderConfig)
}

//保存配置信息，并返回一个日志生成器
func getLogWriter(filename string, maxSize, maxBackup, maxAge int) zapcore.WriteSyncer {
	lumberJackLogger := &lumberjack.Logger{
		Filename:   filename,
		MaxSize:    maxSize,
		MaxBackups: maxBackup,
		MaxAge:     maxAge,
	}
	return zapcore.AddSync(lumberJackLogger)
}

// GinLogger 接收gin框架默认的日志
func GinLogger() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		path := c.Request.URL.Path
		query := c.Request.URL.RawQuery
		c.Next()

		cost := time.Since(start)
		zap.L().Info(path,
			zap.Int("status", c.Writer.Status()),
			zap.String("method", c.Request.Method),
			zap.String("path", path),
			zap.String("query", query),
			zap.String("ip", c.ClientIP()),
			zap.String("user-agent", c.Request.UserAgent()),
			zap.String("errors", c.Errors.ByType(gin.ErrorTypePrivate).String()),
			zap.Duration("cost", cost),
		)
	}
}

// GinRecovery recover掉项目可能出现的panic，并使用zap记录相关日志
func GinRecovery(stack bool) gin.HandlerFunc {
	return func(c *gin.Context) {
		defer func() {
			if err := recover(); err != nil {
				// Check for a broken connection, as it is not really a
				// condition that warrants a panic stack trace.
				var brokenPipe bool
				if ne, ok := err.(*net.OpError); ok {
					if se, ok := ne.Err.(*os.SyscallError); ok {
						if strings.Contains(strings.ToLower(se.Error()), "broken pipe") || strings.Contains(strings.ToLower(se.Error()), "connection reset by peer") {
							brokenPipe = true
						}
					}
				}

				httpRequest, _ := httputil.DumpRequest(c.Request, false)
				if brokenPipe {
					zap.L().Error(c.Request.URL.Path,
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
					)
					// If the connection is dead, we can't write a status to it.
					c.Error(err.(error)) // nolint: errcheck
					c.Abort()
					return
				}

				if stack {
					zap.L().Error("[Recovery from panic]",
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
						zap.String("stack", string(debug.Stack())),
					)
				} else {
					zap.L().Error("[Recovery from panic]",
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
					)
				}
				c.AbortWithStatus(http.StatusInternalServerError)
			}
		}()
		c.Next()
	}
}

```

