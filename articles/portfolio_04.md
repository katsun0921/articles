---
title: "Twitter APIを取得| 4 | オリジナルフィードアプリ作成"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang"]
published: true
---

## portfolio_03

前回まではzennとtwitterのそれぞれのサービスで投稿した情報を取得し、jsonで返しましたので今回はこれらを合わせたものを投稿日の降順にjsonで返したいと思います。

### Controller

```go:apis_controller.go
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

  service := c.Query(constants.QueryService)

  rssServices := [...]string{constants.ZENN}

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

**変更したところ**

zennはrssの情報を取得していて、今後もrssサービスを登録することは考えて、serviceがrssかどうかの判定を行なっています。

```go
  rssServices := [...]string{constants.ZENN}

  isRss := false
  for _, rssService := range rssServices {
    if rssService == service {
      isRss = true
      break
    }
  }
```

queryに値があればそれぞれのapiを取得するserviceを実行し、なければ全てのapiを取得する```GetApiAll()```を実行するようにしています。

```go
  if service == constants.TWITTER {
    resApi, err = services.ApisService.GetTwitter()
  } else if isRss {
    resApi, err = services.ApisService.GetRss(service)
  } else {
    resApi, err = services.ApisService.GetApiAll()
  }
```

新しく全てのapiを取得する```GetApiAll()```を作成

1. 全てのapiを取得し配列に格納。
2. そのあと日付データをUnixに変換した数値をもとにsortを行いました。

<https://pkg.go.dev/sort#example-SliceStable>を参考にしています。goにはsortを行うpkgがあったのですんなりできました。

```go:apis_service.go

import (
  "sort"
)

func (api *apisService) GetApiAll() ([]*apis.Api, rest_errors.RestErr) {
  var res []*apis.Api

  tweets, errTwitter := api.GetTwitter()
  if errTwitter != nil {
    return nil, errTwitter
  }
  zenns, errZenns := api.GetRss(constants.ZENN)
  if errZenns != nil {
    return nil, errZenns
    }
  res = append(res, tweets...)
  res = append(res, zenns...)
  sort.SliceStable(res, func(i, j int) bool { return res[i].DateUnix > res[j].DateUnix })
  return res, nil
}
```

## 総括

ここまでで新着のapiの値をsortすることができました。

ただ現状maxで20までなので、それ以上取得することができないので、次回の記事でさらに追加する機能と追加したいと思います。
