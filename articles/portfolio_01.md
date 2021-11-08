---
title: "オリジナルapiまとめアプリ作成_1"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang", "api"]
published: true
---

## 概要

これまでも記事投稿は行っていました。ただtwitter、だったりzennだったり、別のblogだったり、分散していたのでこう言った投稿を一覧できるものを作りたいと思い投稿をまとめるwebアプリを作成しましたので、作成過程をまとめました。

- バックエンドはGolangを使用
- フロントエンドはNextJsを使用

## Golang とgin

Golang のWebフレームワークはginを使用しました。

<https://github.com/gin-gonic/gin>

## ginでREST APIサーバーを作成

今回作成したものは、他サービスからのAPIをまとめて取得するものだったので、データの取得のみ

### バージョン

```go
go version go1.16.4 darwin/amd64
```

### ディレクトリ構成

```bash
├── app
│   ├── application.go
│   └── url_mappings.go
├── controllers
│   ├── apis
│   │   └── apis_controller.go
│   └── ping
│       └── ping_controller.go
├── go.mod
├── go.sum
├── logger
│   └── logger.go
└── main.go
```

## ベーシックなMVCを作成

### main.go

アプリケーションを実行

```go:main.go
package main

import (
  "github.com/katsun0921/portfolio_api/app"
)


func main() {
  app.StartApplication()
}
```

### app ディレクトリ

ginのrouterを起動処理

```go:app/application.go
package app

import (
  "github.com/gin-gonic/gin"
  "github.com/katsun0921/portfolio_api/logger"
)

var(
  router = gin.Default()
)

func StartApplication() {
  mapUrls()

  logger.Info("about to start the application...")
  router.Run(":8081")
}
```

URLによって処理を振り分ける

```go:app/url_mappings.go
package app

import (
  "github.com/katsun0921/portfolio_api/controllers/ping"
)

func mapUrls() {
	router.GET("/ping", ping.Ping)

}
```

### controllers ディレクトリ

app/url_mappings.goから振られたリクエストをserviceにハンドルし、レスポンスを返します。

```go:controllers/ping/ping_controller.go
package ping

import (
  "github.com/gin-gonic/gin"
  "net/http"
)

func Ping(c *gin.Context) {
  c.String(http.StatusOK, "pong")
}
```

これでgo run するとサーバが起動

```bash
go run ./main.go
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /ping                     --> github.com/katsun0921/portfolio_api/controllers/ping.Ping (3 handlers)
{"level":"info","time":"2021-10-31T22:15:18.017+0900","msg":"about to start the application..."}
[GIN-debug] Listening and serving HTTP on :8081
```

サーバーが起動できたので、localhostにGETを送信するとpongが返ってきます。

```bash
curl localhost:8081/ping
pong
```

[次に進む](./portfolio_02.md)
