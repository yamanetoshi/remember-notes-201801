# octopress 復旧

とりあえず、ソースを入手して `bundle install` してみるに `yajl-ruby` の導入に失敗しているとのこと。`ruby` のバージョンを `1.9.3-p448` して確認してみる方向。

## rbenv で云々

とりあえず以下にトライ。

```
$ rbenv install 1.9.3-p448
$ rbenv local 1.3.8-p448
```

でいいのかな。BUILD FAILED って叱られていたのですが、どうも必要なパケジの導入が足りていなかったようです。でも 2.4.2 って導入できてたのですが何かが違うのだろうな。。

とりあえず以下を導入してリトライ。

```
sudo apt-get install build-essential bison openssl libreadline7 \
    libreadline-dev curl git-core zlib1g zlib1g-dev libssl-dev libyaml-dev \
    libsqlite3-0 libsqlite3-dev sqlite3 libxml2-dev libxslt-dev autoconf \
    libc6-dev libncurses6-dev
```

駄目です。どうも古い ruby が依存してる OpenSSL 関連は deprecated らしい。どうも `octopress` も古いのかどうか。

## マテ

とりあえずバックアップできていない markdown を復旧する事を優先します。なんとなく使ってる octopress は古いらしいので色々検討必要なのかどうか。

- 別のに移行
- 最新の octopress に移行
- 色々アレなので Jekyll にする

とりあえず、復旧対応優先で。
