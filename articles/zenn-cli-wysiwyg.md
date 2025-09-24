---
title: Zennの記事をリッチエディタで！(Zenn CLI 対応)
type: tech
topics:
  - zenn
  - zennfes2025free
  - Tiptap
emoji: 🔖
published: false
---
Zenn で記事を執筆する際はどのエディタを使っていますか？
Web エディタか Zenn CLI が多いと思います。

僕は今まで Web エディタが好きで利用していたのですが、以下の観点が気になり始めました。

- マークダウンと表示の切り替えにタイムラグがあり、スクロール位置も合わない

- 文章が長くなると編集したい箇所を探すのが大変になってくる

- マークダウンと表示の対応関係がわかりずらい

Zenn CLI だとある程度改善されますが、根本的には解決されません。

そこで Notion ライクに執筆したいこともあり、 Zenn CLI に機能拡張という形で WYSIWYG エディタを開発しました。

本記事ではその機能と関連技術について解説します！

## 開発したもの

![WYSIWYGエディタの動作](/images/zenn-cli-wysiwyg/e75dec27-ba7b-48d0-9d66-3d74a9c11d57.gif)

- Zenn CLI 対応：https://github.com/karintou8710/zenn-editor-wysiwyg

- web 版（エディタのお試し用）：https://zenn-wysiwyg-editor.karintou.dev/

成果物のまま編集可能な WYSWIYG エディタで、Zenn の記事を執筆できます！

このエディタは、Zenn CLI というローカルでマークダウンファイルの作成やプレビューができるツールの拡張という形で開発しています。
なので、Zenn CLI の利用感を損なわずに WYSIWYG エディタを活用することが可能です。

エディタのお試し用に Web 版もありますが、マークダウン貼り付けや Git 管理が出来ないなど、実用面で少し問題があるため Zenn CLI 版をおすすめします。

https://zenn.dev/zenn/articles/zenn-cli-guide

### 始め方

[Zenn CLI と同じ方法](https://zenn.dev/zenn/articles/install-zenn-cli)で始められます。
異なる点は、`zenn-cli-wysiwyg` パッケージをインストールすることです。

```bash:zenn-cli-wysiwygの始め方
# 適当な空ディレクトリに移動する
npm init -y
npm install zenn-cli-wysiwyg
npx zenn init
npx zenn
```

:::message alert
**Node v20 以上**をサポートしています。

v20 未満ではエラーになるためご注意ください。
:::

---

（皆さんのスターがモチベーションになるのでぜひ！！）

https://github.com/karintou8710/zenn-editor-wysiwyg

## 機能紹介

Zenn CLI 版は**編集モード**が追加されており、記事画面の**スイッチで切り替え**ができます。

また、**現在は数式以外の Zenn 記法に対応**しています。
（表示のみ対応していて、編集はマークダウンからのみ可能な記法もあります）

以下では、いくつか機能をピックアップして紹介します。

### リアルタイムでマークダウンファイルと同期

![リアルタイム編集](/images/zenn-cli-wysiwyg/1f897ce3-9831-4cd2-96aa-6ced4eaaf50f.gif)

**目玉機能です！**
編集モードでの更新は**リアルタイムでマークダウンに変換**され、ファイルと同期されます。

もちろん、逆のマークダウン → エディタ も対応しています。

### 埋め込み要素の URL 貼り付け

![埋め込み貼り付け](/images/zenn-cli-wysiwyg/faebe6eb-141f-4767-821e-04a832026aef.gif)

マークダウンでは `@[...](...)` の記法が必要なものでも、URL 貼り付けだけで追加するできます。（マークダウン出力では Zenn 記法に変換されます）

特に SpeakerDeck は iframe から ID を取り出す作業が手間でしたが、本エディタではスライドの URL を貼り付けるだけで、自動的に ID を抽出してくれます。

### 画像ファイルのドロップ・ペースト

![画像ファイルドロップ](/images/zenn-cli-wysiwyg/c52692c2-4908-4870-8461-e0d313634ba5.gif)

Zenn CLI では画像のアップロードが面倒なことの 1 つでしたが、WYSIWYG エディタではそれを解決しています。

ドロップ・ペーストされた画像ファイルは `images/<slug>/<uuid>.<ext>` に保存されます。

### スラッシュコマンド

![スラッシュコマンド](/images/zenn-cli-wysiwyg/ffe2f3a2-2c48-4791-a13a-8edf874061df.gif)

Notion のようにスラッシュコマンドにも対応しています。
マークダウン記法を知らなくても、各種ノードを作成することが可能です。

---

詳細の機能は以下の記事をご確認ください！

https://zenn.dev/karintou/articles/eabe0354fcc947

## 技術

[zenn-editor ](https://github.com/zenn-dev/zenn-editor)をフォークして開発をしています。主な変更内容は以下です。

- zenn-cli に Web 編集モードを追加

- WYISWYG エディタ本体の開発

- zenn-markdown-html をブラウザで実行可能に

### zenn-cli に Web 編集モードを追加

zenn-cli は、フロントエンドとバックエンドの構成になっています。
マークダウンファイルを更新すると、リアルタイムでフロントエンドにも反映されてプレビューがやりやすくなっていました。

今回は WYSIWYG エディタと連携するにあたって、逆方向の通信を追加しています。
具体的には WYSIWYG エディタで編集をすると、リアルタイムでマークダウンに変換されてファイルに保存されるようにしました。

この方法では、マークダウン → プレビュー で活用されていた **WebSocket** を採用しています。

### WYISWYG エディタ

zenn-editor はモノレポだったため、エディタで１つのパッケージを作りました。

https://github.com/karintou8710/zenn-editor-wysiwyg/tree/main/packages/zenn-wysiwyg-editor

リッチテキストエディター（RTE）を柔軟に構築できる [TIptap](https://tiptap.dev/) を採用しています。

Tiptap は流行りのヘッドレスなため、UI のカスタマイズ性が非常に高いです。今回のように、Zenn がコンテンツの CSS を提供してくれている場合にはうってつけです。

また Tiptap はドキュメントが豊富でコードも読みやすいため、RTE の中では参考にできるものが多いと思いました。最悪、ラップ元の ProseMirror の関連コードを読んで解決できるという安心感があります。
ProseMirror の方で [Discussion](https://discuss.prosemirror.net/) が活発に動いているため、こちらを参考にすることも多かったです。

#### 独自ノードの作り方

zenn-markdown-html が出力する HTML を参考に、コンテンツの種類・parseHTML・renderHTML を指定します。

例えば、以下はメッセージノードの HTML です。

```html
<aside class="msg message">
  <span class="msg-symbol">!</span>
  <div class="msg-content">
    <p data-line="205" class="code-line">Text</p>
    <p data-line="205" class="code-line">Text</p>
  </div>
</aside>
```

外側の aside が ラッパー になっており、msg-symbol は装飾、msg-content は複数のブロック要素を含みます。基本的に、タグとノードは１：１になります。

msg-symbol は装飾向けのノードのため、別途プラグインで**デコレーション**として追加します。

addNodeView や 通常のノードにすると、キャレットの移動が出来なくなったり、削除可能になったりと色々バグが起きるため、編集可能文書内の装飾は デコレーションにする必要があります。

この、ラッパー・装飾・コンテンツをモデル定義に反映すると、以下のようになります。

```ts:message.ts
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

```ts:message-content.ts
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

```ts:decoration.ts
import type { Node } from '@tiptap/pm/model';
import { Plugin, PluginKey } from '@tiptap/pm/state';
import { Decoration, DecorationSet } from '@tiptap/pm/view';

export function createMessageSymbolDecorationPlugin(nodeName: string) {
  function getDecorations(doc: Node): DecorationSet {
    const decorations: Decoration[] = [];

    doc.descendants((node: Node, pos: number) => {
      if (node.type.name === nodeName) {
        decorations.push(
          Decoration.widget(pos + 1, () => {
            const element = document.createElement('span');
            element.className = 'msg-symbol';
            element.textContent = '!';
            return element;
          })
        );
      }
    });

    return DecorationSet.create(doc, decorations);
  }

  return new Plugin({
    key: new PluginKey('messageSymbolDecoration'),
    state: {
      init(_, { doc }) {
        return getDecorations(doc);
      },
      apply(tr, oldDecorations) {
        if (!tr.docChanged) {
          return oldDecorations.map(tr.mapping, tr.doc);
        }

        return getDecorations(tr.doc);
      },
    },
    props: {
      decorations(state) {
        return this.getState(state);
      },
    },
  });
}
```

これがノードの基礎部分になります。

ここに Backspace などの特殊キーを入力した時の挙動や、マークダウン記法などを機能拡張していくことでノードを構築します。

#### マークダウンとの相互変換

本サービスの文書には３種類のデータ形式があります。

- マークダウン

- 表示用 HTML (zenn-markdown-html でレンダリングしたもの)

- 編集用 HTML (装飾を消して Tiptap で読み込み可能にしたもの)

  - parseHTML と renderHTML は編集用 HTML を扱う

`マークダウン → 編集用 HTML` は、一度 zenn-markdown-html で表示用 HTML にしてから、装飾を DOM 操作で削除して、編集用 HTML に変換しています。
ここで装飾を消さないと、装飾部分がテキストとして認識されて読み込みがバグります。

`編集用 HTML → マークダウン`は、[prosemirror-markdown](https://github.com/ProseMirror/prosemirror-markdown) で変換しています。内部的には、ノードツリーを再起的に辿って、マークダウンを出力しています。

#### 自動テスト

PR Times さんのテスト戦略を参考に、Vitest + Vitest Browser Mode で自動テストを書いています。

https://developers.prtimes.jp/2025/02/20/press-release-editor-frontend-testing-tips/

本プロジェクトでは、キー入力やペーストなどのユーザー操作が伴うものは Vitest Browser Mode、それ以外は Vitest です。

特徴的なこととして、キー入力のテストでは選択の初期位置を `setTextSelection()` などで決めたいことがあります。
しかし、これは内部的に非同期なため以下のコードは失敗します。

```ts
editor.chain().setTextSelection(2).run();
await userEvent.keyboard("a"); // setTextSelection が反映されていない
```

そこで、選択範囲が現在の位置から変わるまでポーリングする `waitSelectionChange` という関数を自作し、以下のようにして解決しています。

```ts
// 現在の選択位置から変化するまで待機する
await waitSelectionChange(() => {
  editor.chain().setTextSelection(2).run();
});

await userEvent.keyboard("a");
```

## まとめ

誇張抜きでめっちゃ使いやすいので、おすすめです！
個人開発は自分が普段から使うものを作るという信念でしたが、これから Zenn の記事はこの WYSIWYG エディタで書くという気持ちになる品物を作れたと思います。
Notion with マークダウン記法で普段書いている人との相性は抜群です。

今は zenn-editor をフォークして開発する形ですが、最終的には本家の zenn-editor にマージをもらえるように、完成度を高めていきます！

実際に皆さんに使っていただいて感想をいただけるとモチベーションになるので、ぜひ！
