# Vite+React+TypeScriptの環境構築用テンプレート

Vite+React+TypeScriptで始める環境のテンプレート

oxlint, oxfmt, stylelintインストール済みで、コミットすると自動でリントと整形を行う。

## リポジトリ作成

右上の「Use this template」から「create a new repository」を押してリポジトリ作成

## miseでnodeのインストール

1. `mise list nodeで`インストールしたnodeのバージョン一覧を確認
2. `mise use node@[バージョン番号]`でローカルにnodeをインストール

## miseでpnpmのインストール

1. `mise list pnpmで`インストールしたpnpmのバージョン一覧を確認
2. `mise use pnpm@[バージョン番号]`でローカルにpnpmをインストール

## `node_modules`のインストール

リポジトリが作れたら`ghq get username/repository`でローカルに`clone`し、`npm`/`pnpm`で`node_modules`をインストールする

## npmパッケージのアップデート

アップデートを確認

```bash
ncu
```

アップデート実行

```bash
ncu -u
npm i
```
