---
title: "RSSからREST APIを返す | オリジナルフィードアプリ作成_2"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang", "api", "gin"]
published: true
---

## rss apiからREST ApiとしてJSON値を取得しレスポンスを返す

前回の続き[ベーシックなMVCを作成](./portfolio_01.md)から今度はzennのrssから任意のJSONを返しました。

**※ 作成中の部分もあり、正常系のテストとかも書いていないので途中でコードが変わるかもしれません。**

### ディレクトリ

```bash
.
├── LICENSE
├── README.md
├── app
│   ├── application.go
│   └── url_mappings.go
├── controllers
│   ├── apis
│   │   └── apis_controller.go
│   └── ping
│       └── ping_controller.go
├── domain
│   └── apis
│       ├── api_dao.go
│       └── api_dto.go
├── go.mod
├── go.sum
├── main.go
└── services
    └── apis_service_.go
```

### app ディレクトリ

新しくRSSのAPI取得するエンドポイントとAPI Controllerを指定

```go:url_mappings.go
package app

import (
  "github.com/katsun0921/portfolio_api/controllers/apis"
  "github.com/katsun0921/portfolio_api/controllers/ping"
)

func mapUrls() {
  router.GET("/ping", ping.Ping)
  router.GET("/apis", apis.Get)
}
```

### controllers ディレクトリ

/apis でリクエストをapi servicesにハンドルをおこない、JSONで返す

```go:controllers/apis/apis_controller.go
package apis

import (
  "errors"
  "github.com/gin-gonic/gin"
  "github.com/katsun0921/go_utils/logger"
  "github.com/katsun0921/go_utils/rest_errors"
  "github.com/katsun0921/portfolio_api/domain/apis"
  "github.com/katsun0921/portfolio_api/services"
  "net/http"
)

func Get(c *gin.Context) {
  var api apis.Api

  rss, err := services.ApisService.GetApi(api)
  if err != nil {
    logger.Error("error when trying to api request", err)
    restErr := rest_errors.NewBadRequestError("invalid json error", errors.New("json error"))
    if restErr != nil {
      return
    }
  }
  c.JSON(http.StatusOK, rss)
}
```

### domain ディレクトリ

データベースとのやりとりをおこなうディレクトリ。今回は外部のapi にアクセスするディレクトリにしました。

rssのdao(Data Access Object)。DBがあればDBのアクセス処理を記述するところ。今回はRSS APIにアクセスを記述しています。

```go：domain/apis/api_dao.go
package apis

import (
  "errors"
  "github.com/katsun0921/go_utils/logger"
  "github.com/katsun0921/go_utils/rest_errors"
  "github.com/mmcdole/gofeed"
)

const feedZenn = "https://zenn.dev/katsun0921/feed"

func (api *Api) GetRss() (*gofeed.Feed, rest_errors.RestErr) {
  feed, err := gofeed.NewParser().ParseURL(feedZenn)
  if err != nil {
    logger.Error("error when trying to rss", err)
    return nil, rest_errors.NewInternalServerError("error when trying to get rss api", errors.New("response error"))
  }

  return feed, nil
}
```

dto(Data Transfer Object)。API からのデータベースから取得した値を格納します。

```go:domain/apis/api_dto.go
package apis

type Api struct {
  Title       string      `json:"title"`
  Description Description `json:"description"`
  Link        string      `json:"link"`
  Service     string      `json:"service"`
  Type        string      `json:"type"`
  DateCreated string      `json:"date_created"`
}

type Description struct {
  PlainText string `json:"plain_text"`
  Html      string `json:"html"`
}
```

### services ディレクトリ

アプリケーション固有の処理やルールが記述される。
取得したapiの検証、保存を実装しています。

```go:services/apis_service_.go
package services

import (
  "github.com/katsun0921/go_utils/rest_errors"
  "github.com/katsun0921/portfolio_api/domain/apis"
  "strings"
)

const (…
  RSS  string = "RSS"
  ZENN string = "zenn"
)

var (
  ApisService apisServiceInterface = &apisService{}
)

type apisServiceInterface interface {
  GetApi(api apis.Api) (*apis.Api, rest_errors.RestErr)
}

type apisService struct {
}

func (*apisService) GetApi(api apis.Api) (*apis.Api, rest_errors.RestErr) {
  result := &apis.Api{}
  feed, err := result.GetRss()
  if err != nil {
    return nil, err
  }

  items := feed.Items
  for _, item := range items {
    itemPlainText := item.Description
    itemPlainText = strings.ReplaceAll(itemPlainText, " ", "")
    itemPlainText = strings.ReplaceAll(itemPlainText, "\n", "")
    result.Title = item.Title
    result.Description.PlainText = itemPlainText
    result.Description.Html = item.Description
    result.Link = item.Link
    result.DateCreated = item.Published
    result.Type = RSS
    result.Service = ZENN
  }

  return result, nil
}

```

## JSONを表示

```bash
curl --location --request GET 'localhost:8081/apis' | jq

{
  "title": "オリジナルフィードアプリ作成_1",
  "description": {
    "plain_text": "概要さまざまなwebサービスに投稿をまとめるAPIを作成しましたので、作成過程をまとめました。簡単な自分だけのフィードアプリです。バックエンドはGolangを使用フロントエンドはNextJsを使用GolangとginGolangのWebフレームワークはginを使用しました。https://github.com/gin-gonic/ginginでRESTAPIサーバーを作成今回作成したものは、他サービスからのAPIをまとめて取得するものだったので、データの取得のみバージョンgoversion...",
    "html": "\n 概要\nさまざまなwebサービスに投稿をまとめるAPIを作成しましたので、作成過程をまとめました。\n簡単な自分だけのフィードアプリです。\n\nバックエンドはGolangを使用\nフロントエンドはNextJsを使用\n\n\n Golang とgin\nGolang のWebフレームワークはginを使用しました。\nhttps://github.com/gin-gonic/gin\n\n ginでREST APIサーバーを作成\n今回作成したものは、他サービスからのAPIをまとめて取得するものだったので、データの取得のみ\n\n バージョン\n\n      \n        \n        go version..."
  },
  "link": "https://zenn.dev/katsun0921/articles/portfolio_01",
  "service": "zenn",
  "type": "RSS",
  "date_created": "Sun, 31 Oct 2021 13:48:27 GMT"
}
```
