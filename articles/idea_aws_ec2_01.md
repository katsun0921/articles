---
title: "AWSのEC2デフォルトシェルをbashからzshに変更する"
emoji: "😸"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["aws", "zsh", "ec2"]
published: false
---

## ZSHをインストール

ローカル環境だとzshだったが、AWSではbashでコマンドの予測変換が使いづらいな〜と感じていたのでzshに変換しました。

ただいろいろ面倒だったので、記録しました。

すでにec2インスタンスがあり、```ssh -i xxx.pem ec2-user@ip-address``` で、sshできている前提のメモになります。

### zshをインストール

インストールは簡単。yumでインストール。

```bash
# amazon linux2 でインスタンスを作成した場合になります。
sudo yum install zsh
cat /etc/shells
```

```bash
# こちらが新しく追加されてます。
/usr/bin/zsh
/bin/zsh
```

```bash
zsh # これでzshが使えるはずです。
```

:::message
おそらく最初は設定画面が出てきます。
:::

(0)を選択する。

```bash
This is the Z Shell configuration function for new users,
zsh-newuser-install.
You are seeing this message because you have no zsh startup files
(the files .zshenv, .zprofile, .zshrc, .zlogin in the directory
~).  This function can help you with a few settings that should
make your use of the shell easier.

You can:

(q)  Quit and do nothing.  The function will be run again next time.

(0)  Exit, creating the file ~/.zshrc containing just a comment.
     That will prevent this function being run again.

(1)  Continue to the main menu.

--- Type one of the keys in parentheses ---
```

### Preztoをインストール

私はローカル環境でPreztoを使用しているので、インストールしました。

ホームディレクトリに.zshrcファイルがあるので削除。

```bash
cd ~/
rm .zshrc
```

Preztoをインストール。***zshを使っている状態です。***

```bash
cd ~/
git clone --recursive https://github.com/sorin-ionescu/prezto.git "${ZDOTDIR:-$HOME}/.zprezto"
setopt EXTENDED_GLOB
for rcfile in "${ZDOTDIR:-$HOME}"/.zprezto/runcoms/^README.md(.N); do
  ln -s "$rcfile" "${ZDOTDIR:-$HOME}/.${rcfile:t}"
done
```

Preztoの設定ファイルが作成されていることを確認。

```bash
.zlogin -> /home/username/.zprezto/runcoms/zlogin
.zlogout -> /home/username/.zprezto/runcoms/zlogout
.zpreztorc -> /home/username/.zprezto/runcoms/zpreztorc
.zprofile -> /home/username/.zprezto/runcoms/zprofile
.zshenv -> /home/username/.zprezto/runcoms/zshenv
.zshrc -> /home/username/.zprezto/runcoms/zshrc
```

zshを再起動する。

```bash
exit
zsh
```

公式サイトのPreztoのテーマになっていればOK！

<https://github.com/sorin-ionescu/prezto#themes>

## ログインのシェルを変更する

せっかくzshをインストールしたが、sshのログイン時はbashなので、```zsh```コマンドを入力しないとzshに変わらない。

これは手間なので、デフォルトシェルをzshに変更しました。

:::message
chshコマンドで切り替えますが、Amazon Linux 2はデフォルトでchshが使えないので、最初にインストールします。
:::

```bash
sudo yum install util-linux-user
```

chshがあることを確認。

```bash
which chsh
/usr/bin/chsh
```

chshが使えるようになったら、シェルをzshに変更する。

```bash
zshのパスを確認する
cat /etc/shells
sudo chsh ec2-user
# フルパスを入力する
/bin/zsh
```

これで切り替わったはずなので、いったん再度ログインして、zshになっていたら成功です。
