---
title: "Twitter APIを取得| 4 | オリジナルフィードアプリ作成"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang"]
published: true
---

## 作業内容

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

## レスポンス

レスポンスを確認してみると、zennのfeedとtwitter の値をそれぞれ返していることを確認できました。

```bash
curl --location --request GET 'localhost:8081/apis' | jq
[
    {
        "text": "Twitter APIを取得| 4 | オリジナルフィードアプリ作成\nportfolio_03前回まではzennとtwitterのそれぞれのサービスで投稿した情報を取得し、jsonで返しましたので今回はこれらを合わせたものを投稿日の降順にjsonで返したいと思います。Controllerapis_controller.gopackageapisimport(\"errors\"\"github.com/gin-gonic/gin\"\"github.com/katsun0921/go_utils/logger\"\"github.com/katsun0921/go_utils...",
        "link": "https://zenn.dev/katsun0921/articles/portfolio_04",
        "service": "zenn",
        "date_created": "2021/11/28 00:00",
        "date_unix": 1638060712
    },
    {
        "text": "RT @goto_nikkei: ◆リスクオフ 一色南アフリカでコロナの変異ウイルスがみつかったことで金融市場は全面的なリスクオフに。日経平均は747円安となりましたが、夕方には先物がさらに200円以上下落。ドル円は113円台に。🇺🇸株先物やNY原油、Bitcoinも大幅安と…",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/26 21:00",
        "date_unix": 1637962220
    },
    {
        "text": "@UdemyJapan ブラックフライデーは、明日まで今回も2つほどWebアプリの講座を購入しました!積み講座にならずきちんと受講しないと",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/25 13:00",
        "date_unix": 1637848449
    },
    {
        "text": "次はページ制作！",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/23 02:00",
        "date_unix": 1637635358
    },
    {
        "text": "@figmadesign いままでXdを使っていたがなんか有償になってしまったので、今回新しくFigmaを使用してみました。ブラウザでさくさく動くので使いやすい！オリジナルなカラーパレットの機能があるといいはどこかにあるのかな？",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/23 02:00",
        "date_unix": 1637635311
    },
    {
        "text": "#webデザイン #figma Figmaで個人のポートフォリオサイトを作ってみました。まだまだ制作途中のものもありますが、いったん形にはなったのでpublishCheck out Portfolio by Katsumas…",
        "link": "https://t.co/0G2Kg8f8aw",
        "service": "twitter",
        "date_created": "2021/11/23 02:00",
        "date_unix": 1637635200
    },
    {
        "text": "RT @goto_nikkei: ◆パウエル続投へバイデン大統領がパウエルFRB議長を続投させる考えだと米メディアが伝えました。現在の任期は来年2月までで、パウエル氏の続投かブレイナードFRB理事の昇格が取り沙汰されていました。上院の承認を経て、正式に就任が決まります",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/22 14:00",
        "date_unix": 1637591652
    },
    {
        "text": "#金融所得課税 ＃格差拡大＃増税＃岸田政権 投資家じゃない方々も一度視聴することおすすめ！いかに岸田政権が格差是正とはかけ離れて、増税したいがために都合の良いデータしか見せていないことがわかります岸田政権がヤバすぎた…",
        "link": "https://t.co/eVteVNrBgw",
        "service": "twitter",
        "date_created": "2021/11/22 13:00",
        "date_unix": 1637589152
    },
    {
        "text": "RT @CyberBrainFish: javascript時計。",
        "link": "https://t.co/7CReyIl4CZ",
        "service": "twitter",
        "date_created": "2021/11/22 13:00",
        "date_unix": 1637586533
    },
    {
        "text": "RT @jingbay: 今度はPythonのPyPiにて11のマルウェア入りパッケージが発見された。backdoorを挿入したりDiscordのaccess tokenを盗んだりと様々。著名なpackageと同名でversionが高いinternal private pa…",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/21 07:00",
        "date_unix": 1637479675
    },
    {
        "text": "RT @watch_UNEXT_A: 🍁#観る読むハマる！秋アニメ🍁🗡️#最果てのパラディン🗡️#UNEXT でアニメもマンガも楽しもう✨5名に電子コミック全8巻セット🎁1️⃣@watch_UNEXT_Aフォロー2️⃣この投稿をRT✅11/26(金)まで規約…",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/20 04:00",
        "date_unix": 1637382028
    },
    {
        "text": "#IKEAのサメ 気になるが、今の公開情報ではなんとも24日の公開まちかな",
        "link": "https://t.co/GDL93FFK2w",
        "service": "twitter",
        "date_created": "2021/11/19 06:00",
        "date_unix": 1637303417
    },
    {
        "text": "#udemy こちらの記事でご紹介されているNext.js×Tailwind CSSで作るモダン開発は、現在udemyでセール価格で販売されているのでこの機会によいかも",
        "link": "https://t.co/BvGWZlw1q5",
        "service": "twitter",
        "date_created": "2021/11/19 03:00",
        "date_unix": 1637294313
    },
    {
        "text": "RT @agektmr: YouTubeがPWA化され、プレミアムユーザーはオフラインでのビデオ視聴ができるようになりました",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/19 03:00",
        "date_unix": 1637290917
    },
    {
        "text": "RT @UdemyJapan: 🎁ブラックフライデーセール🎁フォロー＆RTキャンペーン1200円から購入できるセール期間（11月19～26日）中に1️⃣@UdemyJapanをフォロー2️⃣こちらの投稿をRTしてくれた方の中から抽選で10名様に5000円分のUde…",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/19 00:00",
        "date_unix": 1637280392
    },
    {
        "text": "IEがサポート中は使えない😭",
        "link": "https://t.co/tvZCJJcNrO",
        "service": "twitter",
        "date_created": "2021/11/18 20:00",
        "date_unix": 1637266199
    },
    {
        "text": "#ボジョレーナイトボジョレーナイトに参加してきました!ボジョレーヌーらしく深い赤なのにさっぱりした後味。それでいてほのかに香る甘い香り。レバーパテと一緒に飲むと良い感じに酸味が無くなりGood!",
        "link": "https://t.co/tekBEvWiV4",
        "service": "twitter",
        "date_created": "2021/11/18 12:00",
        "date_unix": 1637238715
    },
    {
        "text": "@nandemoganbaru3 apiを叩くをどうやって叩くんだと思っていた自分がいます。😅",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/18 08:00",
        "date_unix": 1637224136
    },
    {
        "text": "#firefox#FrontEnd 最近webアプリ開発でFirefoxが好きになってきた。jsonをプラグイン不要のデフォルトで整形とheader情報を表示してくれ、JavaScriptsのエラーを対象の変数とかを表示してくれ開発が楽！😃",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/18 08:00",
        "date_unix": 1637223318
    },
    {
        "text": "RT @ChivasRegalJP: ／ミズナラで #新しい日本を味わおう キャンペーン開催❗️#シーバスリーガル＼@ChivasRegalJP をフォロー＆投稿をRTで「シーバスリーガル ミズナラ 18年」が当たる🥃✨・宮藤官九郎サイン入りボトル＋おつまみセット…",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/17 09:00",
        "date_unix": 1637141664
    },
    {
        "text": "@tanakahisateru php がない😐",
        "link": "",
        "service": "twitter",
        "date_created": "2021/11/17 04:00",
        "date_unix": 1637124335
    },
    {
        "text": "Twitter APIを取得 | オリジナルフィードアプリ作成_3\ntwitterapiからRESTApiとしてJSON値を取得しレスポンスを返す前回の続きRSSからRESTAPIを返すから今度はtwitterapiから任意のJSONを返しました。※：作成中の部分もあり、正常系のテストとかも書いていないので途中でコードが変わるかもしれません。ディレクトリ├──LICENSE├──README.md├──go.mod├──go.sum└──src├──app│├──application.go│└──url_mappi...",
        "link": "https://zenn.dev/katsun0921/articles/portfolio_03",
        "service": "zenn",
        "date_created": "2021/11/15 01:00",
        "date_unix": 1636939986
    },
    {
        "text": "RSSからREST APIを返す | オリジナルフィードアプリ作成_2\nrssapiからRESTApiとしてJSON値を取得しレスポンスを返す前回の続きベーシックなMVCを作成から今度はzennのrssから任意のJSONを返しました。※作成中の部分もあり、正常系のテストとかも書いていないので途中でコードが変わるかもしれません。ディレクトリ.├──LICENSE├──README.md├──app│├──application.go│└──url_mappings.go├──controllers│├──apis││└──api...",
        "link": "https://zenn.dev/katsun0921/articles/portfolio_02",
        "service": "zenn",
        "date_created": "2021/11/08 02:00",
        "date_unix": 1636336903
    },
    {
        "text": "オリジナルapiまとめアプリ作成_1\n概要これまでも記事投稿は行っていました。ただtwitter、だったりzennだったり、別のblogだったり、分散していたのでこう言った投稿を一覧できるものを作りたいと思い投稿をまとめるwebアプリを作成しましたので、作成過程をまとめました。バックエンドはGolangを使用フロントエンドはNextJsを使用GolangとginGolangのWebフレームワークはginを使用しました。https://github.com/gin-gonic/ginginでRESTAPIサーバーを作成今回作成したものは、他サービスからのAPIをまとめて取得するものだったの...",
        "link": "https://zenn.dev/katsun0921/articles/portfolio_01",
        "service": "zenn",
        "date_created": "2021/10/31 13:00",
        "date_unix": 1635688107
    }
]
```

## 総括

ここまでで新着のapiの値をsortすることができました。

ただ現状maxで20までなので、それ以上取得することができないので、次回の記事でさらに追加する機能と追加したいと思います。
