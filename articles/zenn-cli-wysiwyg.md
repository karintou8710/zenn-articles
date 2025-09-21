---
title: Zenn の記事をWYSIWYGエディタで書きたい！(Zenn CLI 対応)
type: tech
topics: []
emoji: 🔖
published: false
---

Zenn で記事を執筆する際はどの選択肢を取っていますか？恐らく、Web エディターや Zenn CLI が多いと思います。

この方法はマークダウンを編集してプレビューで確認するものですが、私は Notion のような WYSIWYG エディタで編集したい気持ちがありました。

そこで zenn-editor に機能拡張という形で開発したので、機能と関連技術について解説します！

## 開発したもの

![WYSIWYGエディタの動作](/images/zenn-cli-wysiwyg/e75dec27-ba7b-48d0-9d66-3d74a9c11d57.gif)

- zenn-cli 対応：https://www.npmjs.com/package/zenn-cli-wysiwyg

- web 版（お試し用）：https://zenn-wysiwyg-editor.karintou.dev/

zenn-cli と web の 2 つに対応しています。

web はエディタのお試しができる程度なので、本格的に使いたい方は Git 管理も可能な zenn-cli 版がおすすめです。

zenn-cli 版は以下の `zenn-cli-wysiwyg` パッケージをインストールします。
他は [zenn-cli と同じ方法](https://zenn.dev/zenn/articles/install-zenn-cli)で始められます。

```bash:zenn-cli-wysiwyg の始め方
npm init -y
npm install zenn-cli-wysiwyg
npx zenn init
npx zenn
```

:::message alert
**Node v20 以上**をサポートしています。

v20 未満ではエラーになるためご注意ください。
:::

（皆さんのスターがモチベーションになるのでぜひ！！）

https://github.com/karintou8710/zenn-editor-wysiwyg

## 機能紹介

Zenn CLI版は編集モードが追加されており、記事画面でのスイッチで切り替えができます。

ここを ON にしない限り、通常の Zenn CLI と同じように使えます。

![image](/images/zenn-cli-wysiwyg/581a1e49-f725-4336-9e71-44099251d0dc.png)
*編集モードの切り替えスイッチ*

詳細の機能は以下をご確認ください！本記事ではいくつかピックアップして紹介します。

https://zenn.dev/karintou/articles/eabe0354fcc947

### リアルタイムでマークダウンファイルと同期

![ezgif-8af8df4d40f55e](/images/zenn-cli-wysiwyg/81e8f70c-e523-4556-be2c-2fd32dfe4ce2.gif)

編集モードでの更新はリアルタイムでマークダウンに変換され、同期されます。

もちろん、逆のマークダウン → エディタ の方向も対応しています。

### 埋め込み要素の URL 貼り付け

![ezgif-8e644a887bcb29](/images/zenn-cli-wysiwyg/faebe6eb-141f-4767-821e-04a832026aef.gif)

マークダウンでは `@[...](...)` の記法が必要なものでも、URLの貼り付けだけで追加することが可能です。（マークダウン出力ではきちんと Zenn 記法になっています）

特にSpeaker Deck は iframe から ID を取り出す作業が手間でしたが、本エディタではスライドの URL を貼り付けるだけど、自動的に ID を抽出してくれます。

### 画像ファイルのドロップ・ペースト

![ezgif-12559de3938c15](/images/zenn-cli-wysiwyg/c52692c2-4908-4870-8461-e0d313634ba5.gif)

画像ファイルのドラッグ&ドロップとペーストに対応しています。

画像ファイルは `images/<slug>/<uuid>.<ext>` に保存されます。

## 技術構成

[zenn-editor ](https://github.com/zenn-dev/zenn-editor)をフォークして開発をしています。主な変更内容は以下です。

- WYISWYG エディタ本体の開発

- zenn-cli に Web 編集モードを追加

- zenn-markdown-html をブラウザで実行可能に

### WYISWYG エディタ

リッチテキストエディターを柔軟に構築できる TIptap を採用しています。

https://github.com/karintou8710/zenn-editor-wysiwyg/tree/main/packages/zenn-wysiwyg-editor

### zenn-cli に Web 編集モードを追加

### zenn-markdown-html をブラウザで実行可能に

## まとめ

より完成度を高めて、本家の zenn-editor にマージされるのが最終目標です！
