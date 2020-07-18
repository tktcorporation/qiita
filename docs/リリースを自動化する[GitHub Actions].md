# リポジトリのリリース作業を自動化してみる

タグのプッシュを起点に、リリースの作成、zip の添付までを自動化する仕組みを作ったので、載せておきます。

# 環境

今回自動化したのは、下記のような環境。

- Node 12
- TypeScript
- NestJS
- 開発環境では docker-compose を使用

トランスパイル済みのファイル群を zip にまとめて置く構成にしてみました。

# GitHub Actions の設定

はじめに全体構成を提示して、そのあとで各項目について軽く触れていきます。

## 全体構成

```yml
# Action の名前
name: Release

# 実行タイミング
on:
  push:
    tags: ["v*"]

# 実行するJob
jobs:
  # 1つ目のJob(テスト用)
  test:
    name: Test
    runs-on: ubuntu-latest

    # 順に実行されるステップ処理(テスト用)
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Build containers
        run: docker-compose build

      - name: Up a db container
        run: docker-compose up -d

      - name: Test
        run: docker-compose run --rm app "yarn && yarn test"

  # 2つ目のJob(リリース用)
  release:
    name: Release
    needs: test
    runs-on: ubuntu-latest
    env:
      APP_PATH: ./nestjs

    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Setup Node
        uses: actions/setup-node@v1
        with:
          node-version: "12"

      - name: Build
        run: |
          cd ${{ env.APP_PATH }}
          yarn
          yarn build
          rm -rf ./node_modules/
          yarn --production

      - name: Create Artifact
        run: |
          mkdir ./transpiled
          mv ${{ env.APP_PATH }}/package.json ./transpiled/
          mv ${{ env.APP_PATH }}/yarn.lock ./transpiled/
          mv ${{ env.APP_PATH }}/node_modules ./transpiled/
          mv ${{ env.APP_PATH }}/dist ./transpiled/
          zip -r ${{ github.workspace }}/transpiled.zip ./transpiled

      - name: Create Release
        id: create_release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: ${{ github.ref }}
          body: |
            Release note here
          draft: true
          prerelease: false

      - name: Upload Release Artifact
        id: upload_artifact
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ steps.create_release.outputs.upload_url }}
          asset_path: ${{ github.workspace }}/transpiled.zip
          asset_name: transpiled.zip
          asset_content_type: application/zip

```

# Action の名前

```yml
name: Release
```

実行する Action の名前。

# 実行タイミング

```yml
on:
  push:
    tags: ["v*"]
```

`vなんとか` のタグが push されたときに実行される。

# 実行するJob

```yml
jobs:
```

この次に実行する job を定義する。

# 1つ目のJob(テスト用)

```yml
  test:
    name: Test
    runs-on: ubuntu-latest
```

名前をつけて、Ubuntu 環境で job を実行。

# ステップ処理を順に実行する(テスト用)

```yml
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Build containers
        run: docker-compose build

      - name: Up a db container
        run: docker-compose up -d

      - name: Test
        run: docker-compose run --rm app "yarn && yarn test"
```

`steps` の項目が上から順に実行される。

粒度は **step < job < workflow(１ファイル)**

- workflow は並列実行
- job は通常平行実行
- step は上から１つづつ実行

# 2つ目のJob(リリース用)

```yml
  release:
    name: Release
    needs: test
    runs-on: ubuntu-latest
    env:
      APP_PATH: ./
```

`needs` で test の job が完了してからの実行を指定する。

# ステップ処理を順に実行する(リリース用)

## チェックアウト

```yml
    steps:
      - name: Checkout
        uses: actions/checkout@v2
```

コードを Action にチェックアウトする。

## Node 環境を用意

```yml
      - name: Setup Node
        uses: actions/setup-node@v1
        with:
          node-version: "12"
```

node のコマンドが使えるようにする。

## ビルド

```yml
      - name: Build
        run: |
          cd ${{ env.APP_PATH }}
          yarn
          yarn build
          rm -rf ./node_modules/
          yarn --production
```

本番用のビルドを実行する。

## リリース用の zip 作成

```yml
      - name: Create Artifact
        run: |
          mkdir ./transpiled
          mv ${{ env.APP_PATH }}/package.json ./transpiled/
          mv ${{ env.APP_PATH }}/yarn.lock ./transpiled/
          mv ${{ env.APP_PATH }}/node_modules ./transpiled/
          mv ${{ env.APP_PATH }}/dist ./transpiled/
          zip -r ${{ github.workspace }}/transpiled.zip ./transpiled
```

ビルドしたコードで zip を作る。

## リリースの作成

```yml
      - name: Create Release
        id: create_release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: ${{ github.ref }}
          body: |
            Release note here
          draft: true
          prerelease: false
```

`draft` オプションに true を指定して、下書き状態でリリースを作成する。  
そのままリリースしてしまいたい場合はオプションを外す。

## リリースに Asset を配置する

```yml
      - name: Upload Release Artifact
        id: upload_artifact
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ steps.create_release.outputs.upload_url }}
          asset_path: ${{ github.workspace }}/transpiled.zip
          asset_name: transpiled.zip
          asset_content_type: application/zip
```

２つ前のステップで作成した zip を、１つ前のステップで作成したリリースにアップロードする。

# 確認

リリースの下書きが作成されていることが確認できる。

# 公開

下書きを確認して、 **Publish release** で公開。
