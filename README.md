# textlinter

## Setup

利用できる場合は Docker がおすすめ。

### Docker Env

Docker container に入る。

```bash
$ docker-compose run --rm app /bin/bash
```

### Local Env

Node 環境を用意する(node, npm, yarn コマンドが利用できる)

### Common

node_modules のインストール。

```bash
$ yarn
```

## Usage

### Create Docs

`docs/` 以下にファイルを作成していく。

### Lint

lint を実施する。

```bash
$ yarn lint
```




