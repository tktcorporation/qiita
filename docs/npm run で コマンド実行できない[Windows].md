# 環境

- Windows10
- Docker Toolbox

```
$ docker -v
Docker version 18.09.0, build 4d60db472b
```

# やりたいこと

```powershell
$ docker-compose run --rm /bin/bash
```

```bash
$ npm i --no-bin-links
$ npm run eslint
```

みたいなコマンドを使いたい。

# エラー

```
sh: 1: eslint: not found
```

こんな感じで怒られる。

# 原因

`node_modules/.bin` にコマンドのリンクが置かれていないため。
Mac では自動で作成されていたため、[Windows ではシンボリックリンクがはれない問題](https://qiita.com/Y-Kanoh/items/58815aafb7346930f370) が原因なのかもしれない。

# 解決法

node_modules を一度全て削除し、docker を介さずに `npm i` を実行すると `node_modules/.bin` が作成される。

```powershell
> npm i
> docker-compose run --rm /bin/bash
```

```bash
$ npm run eslint
```

とおった。

# おまけ

http://minatooe.hatenablog.jp/entry/2017/12/15/234418

> あとは、Win は管理者権限なら symlink を作れるようになるらしいので、vagrant を動かすコマンドプロンプトを管理者権限で起動すると--no-bin-links をつけなくても大丈夫になるらしい。

とあったのでやってみたができなかった…。
