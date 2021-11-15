---
title: "Twitter APIを取得 | オリジナルフィードアプリ作成_3"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang", "api", "gin"]
published: true
---

## twitter apiからREST ApiとしてJSON値を取得しレスポンスを返す

前回の続き[RSSからREST APIを返す](./portfolio_02.md)から今度はtwitter apiから任意のJSONを返しました。

**※：作成中の部分もあり、正常系のテストとかも書いていないので途中でコードが変わるかもしれません。**

### ディレクトリ

```bash
├── LICENSE
├── README.md
├── go.mod
├── go.sum
└── src
    ├── app
    │   ├── application.go
    │   └── url_mappings.go
    ├── constants
    │   └── constants.go
    ├── controllers
    │   ├── apis
    │   │   └── apis_controller.go
    │   └── ping
    │       └── ping_controller.go
    ├── domain
    │   └── apis
    │       ├── api_dao.go
    │       └── api_dto.go
    ├── main.go
    └── services
        └── apis_service.go
```

### app ディレクトリ

新しくTwitterのAPI取得するエンドポイントとAPI Controllerを指定

前回と同じく```/apis```にし、queryで```?service=twitter```のときにtwitter apiを取得するようにしました。

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

### constants

今回から定数をconstantsファイルにまとめています。

:::message
頭文字を大文字にしないと外部で呼び出せないので注意
:::

```go:src/controllers/apis/apis_controller.go
package constants

const (
  ZENN         = "zenn"
  TWITTER      = "twitter"
  DateLayout   = "2006/01/02 15:00"
  QueryService = "service"
  MaxCount     = 20
)

```

### controllers ディレクトリ

/apis でリクエストをapi servicesにハンドルをおこない、JSONで返す

```?service=twitter```の時にtwitterを取得するserviceにハンドルしています。

```go:controllers/apis/apis_controller.go
package apis

import (
  "errors"
  "github.com/gin-gonic/gin"
  "github.com/katsun0921/go_utils/logger"
  "github.com/katsun0921/go_utils/rest_errors"
  "github.com/katsun0921/portfolio_api/src/constants"
  "github.com/katsun0921/portfolio_api/src/domain/apis"
  "github.com/katsun0921/portfolio_api/src/services"
  "net/http"
)

func Get(c *gin.Context) {
  var resApi []*apis.Api
  var err rest_errors.RestErr

  service := c.Query(constants.QueryService) // ここでquery情報を取得しています

  rssServices := [...]string{constants.ZENN} // rssは他にもありそうなのでここの配列にある場合はrssのserviceにハンドルしています。

  isRss := false
  for _, rssService := range rssServices {
    if rssService == service {
      isRss = true
      break
    }
  }

  if service == constants.TWITTER {
    resApi, err = services.ApisService.GetTwitter()
  } else if isRss {
    resApi, err = services.ApisService.GetRss(service)
  } else {
    resApi, err = services.ApisService.GetApiAll()
  }

  if err != nil {
    logger.Error("error when trying to api request", err)
    restErr := rest_errors.NewBadRequestError("invalid json error", errors.New("json error"))
    if restErr != nil {
      return
    }
  }
  c.JSON(http.StatusOK, resApi)
}

```

### domain ディレクトリ

データベースとのやりとりをおこなうディレクトリ。今回は外部のapi にアクセスするディレクトリにしました。

twitterのdao(Data Access Object)。DBがあればDBのアクセス処理を記述するところ。今回はTwitter APIにアクセスを記述しています。

twitter api取得するため```github.com/dghubble/go-twitter/twitter```と```github.com/dghubble/oauth1```のpackageを追加しています。

またtwitterのAPI KeyやAccess Tokenの情報を.envファイルに記載し、.env情報を取得するため```github.com/joho/godotenv```も追加しています。

```go:domain/apis/api_dao.go
package apis

import (
  "errors"
  "fmt"
  "github.com/dghubble/go-twitter/twitter"
  "github.com/dghubble/oauth1"
  "github.com/joho/godotenv"
  "github.com/katsun0921/go_utils/logger"
  "github.com/katsun0921/go_utils/rest_errors"
  "github.com/katsun0921/portfolio_api/src/constants"
  "github.com/mmcdole/gofeed"
  "os"
  "strconv"
)

func (api *Api) GetTwitterApi() ([]twitter.Tweet, rest_errors.RestErr) {

  // ここで.env情報を取得
  envErr := godotenv.Load()
  if envErr != nil {
    logger.Error("Error loading .env file", envErr)
    return nil, rest_errors.NewInternalServerError("Error env file", envErr)
  }

  twitterApiKey := os.Getenv("TWITTER_API_KEY")
  twitterApiKeySecret := os.Getenv("TWITTER_API_KEY_SECRET")
  twitterAccessToken := os.Getenv("TWITTER_ACCESS_TOKEN")
  twitterAccessTokenSecret := os.Getenv("TWITTER_ACCESS_TOKEN_SECRET")
  twitterUserId := os.Getenv("TWITTER_USER_ID")

  // ここからtwitter apiを取得。取得するためにはtwitter developer(https://developer.twitter.com/en/docs/getting-started)に登録する必要があります。
  // https://www.itti.jp/web-direction/how-to-apply-for-twitter-api/ の記事を参照させていただきました。
  config := oauth1.NewConfig(twitterApiKey, twitterApiKeySecret)
  token := oauth1.NewToken(twitterAccessToken, twitterAccessTokenSecret)
  // http.Client will automatically authorize Requests
  httpClient := config.Client(oauth1.NoContext, token)

  // twitter client
  client := twitter.NewClient(httpClient)

  // Status Show
  toIntTwitterUserId, errUserId := strconv.ParseInt(twitterUserId, 10, 64)
  if errUserId != nil {
    logger.Error("Error loading User Id from .env file", errUserId)
    return nil, rest_errors.NewInternalServerError("twitter server error to user id", errUserId)
  }

  tweets, httpResponse, err := client.Timelines.UserTimeline(&twitter.UserTimelineParams{
    UserID: toIntTwitterUserId,
    Count:  constants.MaxCount,
  })

  if err != nil {
    logger.Error(fmt.Sprintf("twitter server error %d", httpResponse.StatusCode), err)
    return nil, rest_errors.NewInternalServerError(fmt.Sprintf("twitter server error %d", httpResponse.StatusCode), err)
  }
  return tweets, nil
}
```

dto(Data Transfer Object)。API からのデータベースから取得した値を格納します。

前よりコンパクトに修正しています。

```go:domain/apis/api_dto.go
package apis

type Api struct {
  Text        string `json:"text"`
  Link        string `json:"link"`
  Service     string `json:"service"`
  DateCreated string `json:"date_created"`
  DateUnix    int    `json:"date_unix"`
}
```

### services ディレクトリ

アプリケーション固有の処理やルールが記述される。
取得したapiの検証、保存を実装しています。

```go:services/apis_service_.go
package services

import (
  "github.com/katsun0921/go_utils/rest_errors"
  "github.com/katsun0921/portfolio_api/src/constants"
  "github.com/katsun0921/portfolio_api/src/domain/apis"
  "regexp"
  "sort"
  "strings"
  "time"
)

var (
  ApisService apisServiceInterface = &apisService{}
)

type apisServiceInterface interface {
  GetRss(service string) ([]*apis.Api, rest_errors.RestErr)
  GetTwitter() ([]*apis.Api, rest_errors.RestErr)
}

type apisService struct {
}

func (*apisService) GetTwitter() ([]*apis.Api, rest_errors.RestErr) {
  api := &apis.Api{}
  var res []*apis.Api
  tweets, err := api.GetTwitterApi()
  if err != nil {
    return nil, err
  }

  for i := 0; i < constants.MaxCount; i++ {// 20件取得しています
    if i >= len(tweets) {
      break
    }

    key := &apis.Api{}
    tweetText := strings.ReplaceAll(tweets[i].Text, "\n", "")
    regLink := regexp.MustCompile("https:.*$")// リンク情報がTextに含まれているため、正規表現でhttpsから始まるテキストを取得
    tweetPlainText := regLink.ReplaceAllString(tweetText, "")
    tweetPlainText = strings.TrimSpace(tweetPlainText)
    tweetLink := regLink.FindString(tweetText)

    t, _ := time.Parse("Mon Jan 2 15:04:05 MST 2006", tweets[i].CreatedAt)// Time情報を表示するテキストにパース
    tweetDate := t.Format(constants.DateLayout)
    key.Text = tweetPlainText
    key.Link = tweetLink
    key.DateCreated = tweetDate
    key.DateUnix = int(t.Unix())
    key.Service = constants.TWITTER

    res = append(res, key)
  }

  return res, nil
}
```

## JSONを表示

```bash
curl --location --request GET 'localhost:8081/apis?service=twitter' | jq
[GIN] 2021/11/15 - 10:29:14 | 200 |  711.580904ms |             ::1 | GET      "/apis?service=t│
witter" // 500msを超えちゃうので、もう少しコードを考えないといけなさそう

[
    {
        "text": "#KAVALANKAVALANって缶も販売していたのか！",
        "link": "https://t.co/vY1j3qW56D",
        "service": "twitter",
        "date_created": "2021/11/11 09:00",
        "date_unix": 1636621952
    },
    {
        "text": "#unitysoftware 最近の株の値上がりはこれが理由かな",
        "link": "https://t.co/rr5cUq3Qha",
        "service": "twitter",
        "date_created": "2021/11/10 03:00",
        "date_unix": 1636513592
    },
    {
        "text": "#ワイン 京都か〜",
        "link": "https://t.co/1dS0mgoc6k",
        "service": "twitter",
        "date_created": "2021/11/10 03:00",
        "date_unix": 1636513413
    },
    {
        "text": "楽しみ!",
        "link": "https://t.co/Clr9BuOsdh",
        "service": "twitter",
        "date_created": "2021/11/09 03:00",
        "date_unix": 1636428766
    },
]
```
