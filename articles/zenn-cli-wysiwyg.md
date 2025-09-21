---
title: Zenn の記事をWYSIWYGで！(Zenn CLI 対応)
type: tech
topics: []
emoji: 🔖
published: false
---
Zenn で記事を執筆する際はどのエディタを使っていますか？恐らく、Web エディターか Zenn CLI が多いと思います。

これらの方法はマークダウンを編集してプレビューで確認するものですが、私は Notion のような WYSIWYG エディタで編集したい気持ちがありました。

そこで Zenn CLI に機能拡張という形で開発したので、機能と関連技術について解説します！

## 開発したもの

![WYSIWYGエディタの動作](/images/zenn-cli-wysiwyg/e75dec27-ba7b-48d0-9d66-3d74a9c11d57.gif)

- Zenn CLI 対応：https://www.npmjs.com/package/zenn-cli-wysiwyg

- web 版（お試し用）：https://zenn-wysiwyg-editor.karintou.dev/

Zenn CLI と web の 2 つに対応しています。

web はエディタのお試しができる程度なので、本格的に使いたい方は Git 管理も可能な Zenn CLI版がおすすめです。

Zenn CLI 版は以下の `zenn-cli-wysiwyg` パッケージをインストールします。
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

Zenn CLI 版は編集モードが追加されており、記事画面のスイッチで切り替えができます。

ここを ON にしない限り、通常の Zenn CLI と同じように使えます。

詳細の機能は以下をご確認ください！本記事ではいくつかピックアップして紹介します。

https://zenn.dev/karintou/articles/eabe0354fcc947

### リアルタイムでマークダウンファイルと同期

![ezgif-8af8df4d40f55e](/images/zenn-cli-wysiwyg/81e8f70c-e523-4556-be2c-2fd32dfe4ce2.gif)

編集モードでの更新はリアルタイムでマークダウンに変換され、ファイルと同期されます。

もちろん、逆のマークダウン → エディタ の方向も対応しています。

### 埋め込み要素の URL 貼り付け

![ezgif-8e644a887bcb29](/images/zenn-cli-wysiwyg/faebe6eb-141f-4767-821e-04a832026aef.gif)

マークダウンでは `@[...](...)` の記法が必要なものでも、URL の貼り付けだけで追加することが可能です。（マークダウン出力ではきちんと Zenn 記法になっています）

特に SpeakerDeck は iframe から ID を取り出す作業が手間でしたが、本エディタではスライドの URL を貼り付けるだけで、自動的に ID を抽出してくれます。

### 画像ファイルのドロップ・ペースト

![ezgif-12559de3938c15](/images/zenn-cli-wysiwyg/c52692c2-4908-4870-8461-e0d313634ba5.gif)

画像ファイルのドラッグ&ドロップとペーストに対応しています。

画像ファイルは `images/<slug>/<uuid>.<ext>` に保存されます。

### コードブロック

## 技術

[zenn-editor ](https://github.com/zenn-dev/zenn-editor)をフォークして開発をしています。主な変更内容は以下です。

- WYISWYG エディタ本体の開発

- zenn-cli に Web 編集モードを追加

- zenn-markdown-html をブラウザで実行可能に

### WYISWYG エディタ

zenn-editor はモノレポだったため、エディタで１つのパッケージを作りました。

https://github.com/karintou8710/zenn-editor-wysiwyg/tree/main/packages/zenn-wysiwyg-editor

リッチテキストエディター（RTE）を柔軟に構築できる [TIptap](https://tiptap.dev/) を採用しています。

Tiptap は流行りのヘッドレスなため、UI のカスタマイズ性が非常に高いです。今回のように、Zenn がコンテンツの CSS を提供してくれている場合にはうってつけです。

また Tiptap はドキュメントが豊富でコードも読みやすいため、RTE の中では参考にできるものが多いと思いました。最悪、ラップ元の ProseMirror の関連コードを読んで解決できるという安心感があります。
ProseMirrorの方で [Discussion](https://discuss.prosemirror.net/) が活発に動いているため、こちらを参考にすることも多かったです。

:::message
本節のここから先は、ProseMirror と Tiptap がわかる方向けの解説です。
:::

#### 独自ノードの作り方

zenn-markdown-html が出力する HTML を参考に、コンテンツの種類・parseHTML・renderHTML を指定します。

例えば、以下はメッセージノードのHTMLです。

```html
<aside class="msg message">
  <span class="msg-symbol">!</span>
  <div class="msg-content">
    <p data-line="205" class="code-line">Text</p>
    <p data-line="205" class="code-line">Text</p>
  </div>
</aside>
```

外側の aside が wrapper になっており、msg-symbol は装飾、msg-content は複数のブロック要素を含みます。

これをモデル定義に反映すると、以下のようになります。
基本的に、タグとノードは１：１になります。

```ts
export const Message = Node.create({
  name: 'message',
  group: 'block',
  content: 'messageContent',

  parseHTML() {
    return [
      {
        tag: 'aside.msg',
        getAttrs: (element) => {
          return {
            type: element.classList.contains('alert') ? 'alert' : 'message',
          };
        },
      },
    ];
  },

  renderHTML({ node, HTMLAttributes }) {
    return [
      'aside',
      mergeAttributes(HTMLAttributes, {
        class: cn('msg', {
          alert: node.attrs.type === 'alert',
        }),
      }),
      0,
    ];
  },
...
});

```

```ts
import { mergeAttributes, Node } from '@tiptap/react';

export const MessageContent = Node.create({
  name: 'messageContent',
  content: 'block+',

  parseHTML() {
    return [
      {
        tag: 'div.msg-content',
      },
    ];
  },

  renderHTML({ HTMLAttributes }) {
    return [
      'div',
      mergeAttributes(HTMLAttributes, {
        class: 'msg-content',
      }),
      0,
    ];
  },
...
});

```

msg-symbol は装飾向けのノードのため、別途プラグインでデコレーションとして追加します。

addNodeView や 通常のノードにすると、キャレットの移動が出来なくなったり、削除可能になったりと色々バグが起きるため、編集可能文書内の装飾は デコレーションにする必要があります。

#### マークダウンとの相互変換

文書には３種類のデータ形式があります。

- マークダウン

- 表示用 HTML (zenn-markdown-html でレンダリングしたもの)

- 編集用 HTML (装飾を消して Tiptap で読み込み可能にしたもの)

  - parseHTML と renderHTML は編集用 HTML を扱う

マークダウン → 編集用 HTML は、一度 zenn-markdown-html で表示用 HTML にしてから、装飾をDOM操作で削除して、編集用 HTML に変換しています。

編集用 HTML → マークダウンは、[prosemirror-markdown](https://github.com/ProseMirror/prosemirror-markdown) で変換しています。内部的には、ノードツリーを再起的に辿って、マークダウンを出力しています。

#### 自動テスト

### zenn-cli に Web 編集モードを追加

編集モードが ON の時は、 更新時に WebSocket で変更通知を送り、都度ファイルを更新するようにしています。

## まとめ

自分で言うのもアレですが、めっちゃ使いやすいのでおすすめです！

Notion with マークダウン記法で普段書いている人との相性は抜群だと思います。

最終的には本家の zenn-editor にマージをもらえるように、完成度を高めていきます！
