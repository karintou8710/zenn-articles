---
title: Tiptap 独自ノード開発の入門
type: tech
topics: []
emoji: 🔖
published: false
---
Tiptap で公式のチュートリアルで RTE を表示するところまでは割と簡単にできます。

ですが、実際に拡張機能を開発するとなった時には機能と固有の概念が多すぎる故に、何から始めたらいいのか戸惑うかもしれません。（私はかなり戸惑った）

本記事では初めて触れる方でもわかるように、RTE の基本を抑えながら Tiptap で簡単な Zenn の RTE を開発します。

## 本記事の対象読者

- Tiptap で用意された機能の表示はできるが、**拡張機能の作成方法**がわからない

- React の基本がわかる

## 今回開発するもの

Zenn はコンテンツ部分の CSS を [zenn-content-css](https://github.com/zenn-dev/zenn-editor/tree/canary/packages/zenn-content-css) の OSS として公開しています。お手頃にデザインに優れた RTE を開発するために、今回はこちらを利用させていただきます。

実装する機能一覧

- 段落

- 見出し 1 \~ 4

※ 学習のため、Tiptap が提供する拡張は使用しません。

## RTE を状態と操作で捉える

ライブラリを触る前に、RTE がどういった要素から成り立っているのかを考えると、後々の開発でスムーズに入れます。

RTE とはリッチな文書データを見た目のまま編集できるエディタです。
Notion など利用者側の立場になることは多いですが、開発する側になると内部の実装方法がわからないです。
こんな時は、[操作より状態・性質に着目する](https://zenn.dev/knowledgework/articles/c48539d2f35ecc)という記事にあるように、RTE でも同様に考えると理解しやすいです。そうすると、以下のように切り分けができます。

- **状態（文書構造, 選択範囲）**

- 文書への編集操作

  - 指定した位置にテキストを追加・削除するなど

- ユーザーと RTE のインターフェース

  - キーボード入力、マウス操作、ブラウザの表示。。。

成り立つべき状態の性質が決まると、操作は状態の変化になります。インターフェースは主に編集操作のイベントです。
よって、状態について詳しく見ることが RTE への深い理解につながるため解説します。

### 状態 (文書構造)

ブラウザの RTE は HTML を文書構造の状態として保持しています。

```html
<div>
  <p>Text</p>
</div>
```

例えば段落の先頭に `a` を入力すると次のようになります。直感的ですね。

```html
<div>
  <p>aText</p>
</div>
```

文書構造の基本的な状態はこれだけです。その時の HTML を保持しています。

ただ、文書構造の取るべき構造を決めたい時があります。`<p><h1>Heading</h1></p>` といった段落の中に見出しを含めるといった構造は許したくないでしょう。

標準の contenteditable だと取るべき構造を定義する API がないため難しいのですが、最近の RTE ライブラリでは提供されています。詳細は Tiptap の章で解説します。

### 状態（選択範囲）

編集には、キャレットが文書内のどこを指しているのかという情報が必要です。

選択範囲は、Web 標準では [Selection](https://developer.mozilla.org/ja/docs/Web/API/Selection) で管理されています。Selection は複数の [Range](https://developer.mozilla.org/ja/docs/Web/API/Range) を持ちます。Range は範囲選択した時に出る**青い矩形 1 つ**です。

Firefox では複数の Range を持つことができますが、他のブラウザは 1 つのみ対応しています。

Range は、主に 4 つの状態を持っています。

- startContainer, endContainer

- startOffset, endOffset

container は DOM であり、offset は container 内の位置です。start は先頭で、end は末尾を指しています。

```html
<!-- exを範囲選択 -->
<p>T|ex|t</p>
```

上記を範囲選択している状態は、startContainer, endContainer がともに TextNode になります。
そして、`startOffset=1`, `endOffset=3` のように状態が決定します。
もし、範囲選択が閉じている場合は start と end ともに同じものになります。

この話を詳しく知りたい方は、以下の記事がわかりやすいです。

https://ja.javascript.info/selection-range

Web 標準では選択範囲を DOM と オフセットで持っていましたが、これだと文書全体のどこに位置しているのかがわからず、プログラム的にも扱いづらいです。

なので、Tiptap では文書全体で一意な数値が各位置に割り当てられています。詳細は Tiptap の章で説明します。

## Tiptap の概要

https://tiptap.dev/

Tiptap はヘッドレスなリッチテキストエディタです。従来のものは、ライブラリ側である程度スタイルが密結合している状態で提供されていましたが、ヘッドレスなので機能のみです。
カスタマイズ性が非常に高くなっているため、１からオリジナルのエディタを開発する場合に光ります。

シンプルなコア機能に対して必要な機能を拡張する設計で、Tiptap が多くの拡張機能を提供しています。ですが、Tiptap を使った開発をする場合は基本的にサービスに沿った拡張が必要になることが多いので、独自拡張の実装はほぼ必須となるでしょう。

前章を踏まえて、独自拡張を作る上での状態定義を見てみます。

### 状態（文書構造）の定義

Tiptap では文書構造をノード単位で定義します。ノードは**意味のある単位の DOM 構造**で、おおよその場合はノードと DOM が１対１になっています。

ノードに大きく２種類のタイプがあり、**ブロックノードとインラインノード**があります。CSS のイメージと同じです。この区分は、後の位置計算に関わってきます。

ノードに最低限必要な情報は以下です。

- 名前

- **ブロック or インライン**

- **子要素に持てるノードの種類**

- **HTML との入出力インターフェース**

シンプルな**トップノード + 複数段落 + テキスト** の組み合わせを考えてみます。イメージする HTML は次のようなものです。

```html
<div>
  <p>Text</p>
  <p></p>
  <p>Text2</p>
</div>
```

#### 文書構造実装のシンプルな例

まずはトップノードである Document 構造はこれです。

```ts
import { Node } from "@tiptap/core";

export const Document = Node.create({
  name: "doc",
  topNode: true,
  content: "block+",
});
```

Document は特殊なため、細かい箇所は気にする必要はありません。
content にある `block+` だけ重要で、これは group が block であるノードを 1 つ以上子要素として持つということです。
今回は block 要素が段落のみなので、1 つ以上の段落を持つという意味になります。

つまり、`<div></div>` のような中身が何もない HTML は許容されません。

```ts
import { Node } from "@tiptap/core";

export const Paragraph = Node.create({
  name: "paragraph", // 名前
  content: "inline*", // 子に持てるノードの種類
  group: "block", // (他のcontentで指定できるように、利便性のため)

  // デフォルトでブロックタイプ (inline: false)

  parseHTML() {
    return [{ tag: "p" }];
  },

  renderHTML() {
    return ["p", 0];
  },
});
```

content で子要素に持てるノードが inline\* となっていますが、これは group が inline なノードを 0 個以上含めれるという意味になります。テキストノードは inline なので含めれます。

```ts
import { Node } from "@tiptap/core";

export const Text = Node.create({
  name: "text",
  group: "inline",
});
```

parseHTML は HTML の入力を担当しており、p タグを Paragraph ノードに変換してくれます。
renderHTML は出力で、Paragraph ノードを p タグに変換してくれます。

`["p", 0]` の 0 はホールを表す特殊な記法で、子要素の renderHTML を呼び出します。（他にいい方法なかったのかな。。）

:::message
Document と Text は特殊なノードなため、renderHTML や parseHTML は必要ありません。

他にも、ブロック or インラインの判断は **inline プロパティ**で行いますが、Text では無くてもインラインノードとして認識されるっぽいです。（デフォルトはブロック）

group には任意のテキストを指定可能で、block, inline はノードタイプに影響を及ぼさないため注意。
:::

### 状態（選択範囲）

## Zenn RTE の開発

拡張機能の勉強のため、Tiptap が用意しているものは使用しません。

## まとめ

RTE での実装の考え方、流れについて体験していただけたら嬉しいです。

他の機能の実装もみたい。。となった場合、Zenn の WYSIWYG エディタを開発しているので参考にしてもらえると！
本記事を読んだ後であれば、コードの雰囲気が伝わると思います。

https://github.com/karintou8710/zenn-editor-wysiwyg
