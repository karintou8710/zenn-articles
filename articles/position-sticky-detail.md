---
title: 'position: sticky 雰囲気で使ってない？'
type: tech
topics: []
emoji: 🔖
published: false
---
`position: sticky`、なんとなく雰囲気で使ってませんか？自分はその一人でした。

ヘッダーを上に固定するなど単純なケースはコピペで十分だったのですが、ちょっと複雑になるとsticky が思ったように効かないケースがあります。

この時は仕様を知らないと原因調査が難しいため、調べたことを本記事に載せます。

誰かの助けになれば嬉しいです。

## TL;DR

登場人物は sticky 要素・親要素・表示領域（sticky view rectangle）の3つ。

表示領域は、sticky要素に最も近いスクロール可能な祖先要素の領域で、`inset`（top, bottom, left, right）を引いた範囲。ルートの場合は viewport になる。

sticky要素は、スクロール時に親要素の範囲内で、表示領域からはみ出さないように動く。

## 概要

MDN には細かい仕様が載っていなかったので、[CSS Positioned Layout Module](https://drafts.csswg.org/css-position/) の仕様書を読みました。内容が難しかったので、噛み砕いて例を挙げながら解説します。

登場人物は sticky 要素・親要素・表示領域（sticky view rectangle）の 3 つです。
その中で重要なのは以下です。

- sticky 要素

  - `position: sticky` と `inset(top, bottom, left, right)`を指定する

  - 基本は親要素の相対位置で配置される。inset が auto 以外の方向の表示領域超えると、はみ出さないように移動する

  - sticky 要素は親要素をはみ出して動かない

- 表示領域（sticky view rectangle）

  - sticky 要素から一番近いスクロール可能な祖先要素（ルートは viewport）

  - 表示領域の高さは、スクロール要素の高さ − ( top + bottom )

  - 表示領域の横幅は、スクロール要素の横幅 − ( left + right )

言葉で説明しても分かりづらいので、図を用います。

### ケース１：ヘッダーを一番上に固定

```html
<body>
  <header>sticky header</header>
  <div class="box1">Box1</div>
  <div class="box2">Box2</div>
</body>
```

```css
header {
  position: sticky;
  background: orange;
  top: 0;
  height: 100px;
}

.box1 {
  height: 1000px;
  background: gray;
}

.box2 {
  height: 1000px;
  background: tan;
}
```

```mermaid
https://codesandbox.io/embed/tqpt4s?view=editor+%2B+preview&module=%2Findex.html
```

## 複雑なケース

th を viewport（画面全体）に対して sticky かつ テーブルを横スクロールにしたいとという要件があったとします。
この時 thの親要素は当然テーブルなので、そこが横スクロールになることでsticky 表示領域がテーブルになってしまいます。
そうすると、th が viewport に対して粘着するのが不可能になります。

調査してみると、この問題は多くの人が遭遇しているみたいです。

https://github.com/w3c/csswg-drafts/issues/8286

現状だと、sticky-x, sticky-y のように方向を含めて指定することが出来ません。sticky は垂直方向に粘着させたくても、テーブルで overflow-x: auto が指定されていると、そこが sticky 表示領域となってします。

軸ごとの sticky や、表示領域の要素を任意に指定できる記法を望んでいる声がありました。

調査してみると、元のテーブルをそのままに、クローンしたテーブルを position: fixed にして解決するライブラリがありました。

https://github.com/archfz/vh-sticky-table-header

thead を fixed にすると thead がないものとして扱われるため、tbody のセルに横幅などで影響を与えます。なので thead の構造をそのままに、クリックの伝搬をして対応しています。

ただ、アクセシビリティ的にどうなんだろう？みたいなことは思います。

## 参考

https://developer.mozilla.org/ja/docs/Web/CSS/position

https://drafts.csswg.org/css-position
