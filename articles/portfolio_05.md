---
title: "記事をデータベースに格納 | 5 | オリジナルフィードアプリ作成"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang", "MySQL","Migration"]
published: true
---

## データベースに記事を格納しました

追加記事の取得をいろいろ考えてみたところ、今までの記事をデータベースに格納することにして追加記事を取得することにしました。

一時的にファイルを作成してみることも考えてみたのですが、下記の理由からやめました。

- リクエストするたびにファイルを作成するのか
- Twitterは20回までやIDから取得する記事を判別できるが、feedはリクエスト時に全て取得するといったリクエスト時の情報取得が違う
- 新しいサービスを追加するとどうするか

## golang-migrate

環境はMacOSの場合

```bash
brew install golang-migrate
go get -tags 'mysql' -u github.com/golang-migrate/migrate/cmd/migrate
```

### migration用のファイルを作成

公式サイトを見るとupとdownの2つファイルを作成し、ファイル名の頭文字にversionをつけるのがBest practicesみたいです。

[Migrations](https://github.com/golang-migrate/migrate/blob/master/MIGRATIONS.md)

```bash:example
{version}_{title}.up.{extension}
{version}_{title}.down.{extension}

# version
1_initialize_schema.down.sql
1_initialize_schema.up.sql
2_add_table.down.sql
2_add_table.up.sql

# timestamp
1500360784_initialize_schema.down.sql
1500360784_initialize_schema.up.sql
1500445949_add_table.down.sql
1500445949_add_table.up.sql
```

### migrateを実行

とりあえずexampleにあった処理を実行してみる。

upファイルには、testテーブルを作成し、firstnameカラムを追加

downファイルには、testテーブルを削除する

<https://github.com/golang-migrate/migrate/tree/master/database/mysql/examples/migrations>

```sql:1_initialize_schema.down.sql
CREATE TABLE IF NOT EXISTS test (
  firstname VARCHAR(16)
);
```

```sql:1_initialize_schema.up.sql
CREATE TABLE IF NOT EXISTS test (
  firstname VARCHAR(16)
);
```

```bash

# 事前にMySQLにarticlesのDBを作成
migrate -source file://migrations/ -database 'mysql://root:@tcp(127.0.0.1:3306)/articles' up 1

mysql> use articles;
mysql> show tables;
+--------------------+
| Tables_in_articles |
+--------------------+
| schema_migrations  |
| test               |
+--------------------+

mysql> DESC test;
+-----------+-------------+------+-----+---------+-------+
| Field     | Type        | Null | Key | Default | Extra |
+-----------+-------------+------+-----+---------+-------+
| firstname | varchar(16) | YES  |     | NULL    |       |
+-----------+-------------+------+-----+---------+-------+
```

tableとカラムが作成されていました。

続いて削除

```bash
migrate -source file://migrations/ -database 'mysql://root:@tcp(127.0.0.1:3306)/articles' down 1

mysql> show tables;
+--------------------+
| Tables_in_articles |
+--------------------+
| schema_migrations  |
+--------------------+
```

削除されていました。

## まとめ

Golangには、Railsとかの```rails generate``` がなく、sqlファイルでcliを実行するという作業になりました。

戸惑いながらもテーブルとカラムを作成できましたので、次回は必要なカラムとそこに情報を挿入する作業をしたいと思います。
