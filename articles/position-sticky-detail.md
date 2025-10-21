---
title: "position: sticky 雰囲気で使ってない？"
type: tech
topics: ["css", "frontend", "zennfes2025free", "sticky"]
emoji: 📎
published: true
---

`position: sticky`、なんとなく雰囲気で使ってませんか？自分はその一人でした。

ヘッダーを上に固定するなど単純なケースはコピペで十分だったのですが、ちょっと複雑になると sticky が思ったように効かないケースがあります。

この時は仕様を知らないと原因調査が難しいため、調べたことを本記事に載せます。

誰かの助けになれば嬉しいです。

## TL;DR

登場人物は sticky 要素・親要素・sticky 表示領域（sticky view rectangle）の 3 つ。

sticky 表示領域は、sticky 要素に最も近いスクロール可能な祖先要素の領域で、`inset`（top, bottom, left, right）を引いた範囲。ルートの場合は viewport になる。

sticky 要素は、スクロール時に親要素の範囲内で、sticky 表示領域からはみ出さないように動く。

## 概要

MDN には細かい仕様が載っていなかったので、[CSS Positioned Layout Module](https://drafts.csswg.org/css-position/) の仕様書を読みました。内容が難しかったので、噛み砕いて例を挙げながら解説します。

登場人物は sticky 要素・親要素・sticky 表示領域（sticky view rectangle）の 3 つです。
その中で重要なのは以下です。

- sticky 要素

  - `position: sticky` と `inset(top, bottom, left, right)`を指定する

  - 基本は親要素の相対位置で配置される。

  - inset が auto 以外の方向の sticky 表示領域超えると、はみ出さないように移動する

  - sticky 要素は親要素をはみ出して動かない

- sticky 表示領域（sticky view rectangle）

  - **sticky 要素から一番近いスクロール可能な祖先要素（ルートは viewport）**

  - sticky 表示領域の高さは、スクロール要素の高さ − ( top + bottom )

  - sticky 表示領域の横幅は、スクロール要素の横幅 − ( left + right )

言葉で説明しても分かりづらいので、図を用います。

### 例：ヘッダーを上に固定

```html
<body>
  <header class="sticky">sticky header</header>
  <div class="box1">Box1</div>
  <div class="box2">Box2</div>
</body>
```

```css
.sticky {
  position: sticky;
  top: 0;
  height: 100px;
}

.box1 {
  height: 1000px;
}

.box2 {
  height: 1000px;
}
```

@[codesandbox](https://codesandbox.io/embed/tqpt4s)

![image](/images/position-sticky-detail/2952878b-be3c-4135-aa4a-a4eb0c0479c6.png)
_初期表示_

左から順に、**viewport(白) ・sticky 表示領域(紫)・sticky 要素の親(黄色)** になります。
viewport はディスプレイで表示されてる領域です。

今回は top が指定されているので、以下のイメージで sticky 要素が動きます。

- **紫と黄色の重なった領域から sticky 要素が上にはみ出ている場合、そこの一番上になるように移動させる**

今回は top: 0 のみ指定されているため、viewport と sticky 表示領域が同じになります。
また、sticky 表示領域は viewport と連動して動きます。
親要素である body は描画された領域全体のため、 sticky 要素は全体を動くことができます。

初期では sticky 表示領域内に sticky 要素がおさまっているため、相対配置と変わりません。

少し下にスクロールして動かして見ましょう。

![image](/images/position-sticky-detail/5f2023e6-cc94-47aa-b645-83ac38314a0a.png)
_下にスクロール_

下にスクロールすることで、sticky 要素は紫と黄色の重なった部分に収まるように移動します。

この時、画面全体の高さは変わらないことに注意してください。元の位置に同じサイズの空要素を配置して移動しているイメージです。

上の例は top: 0 でしたが、top: 100px の場合は次のようになります。

@[codesandbox](https://codesandbox.io/embed/ttqnd9)

![image](/images/position-sticky-detail/7a65a76b-8cb0-4202-ae6f-b099914ee0f2.png)
_top: 100px (ヘッダーの高さ)_

top: 100px を指定したことで、sticky 表示領域の上部の 100px が制限されます。

よって、viewport の上 100px に空白ができて、その下に sticky 要素が粘着します。

---

親要素の高さが固定の場合は、図の右側の黄色い範囲が制限になります。

## 複雑なケース

テーブルヘッダーを viewport（画面全体）に対して sticky かつ テーブルを横スクロールにしたいとという要件があったとします。
この時 thead の親要素は当然テーブルなので、そこが横スクロールになることで sticky 表示領域がテーブルになってしまいます。
そうすると、thead が viewport に対して粘着するのが不可能になります。

調査してみると、この問題は多くの人が遭遇しているみたいです。

https://github.com/w3c/csswg-drafts/issues/8286

現状だと、sticky-x, sticky-y のように方向を含めて指定することが出来ません。sticky は垂直方向に粘着させたくても、テーブルで overflow-x: auto が指定されていると、そこが sticky 表示領域となってします。

軸ごとの sticky や、表示領域の要素を任意に指定できる記法を望んでいる声がありました。

調査してみると、元のテーブルをそのままに、クローンしたテーブルを position: fixed にして解決するライブラリがありました。

https://github.com/archfz/vh-sticky-table-header

table の構造をそのままに thead を コピーした要素を隣に作成し、position: fixed を JS で制御して sticky 構造を実装しています。
クリックどうするんだろう？と思って調べてみたら、伝搬していました。

## 最後に

完全に理解したけど、何もわからん。

## 参考

https://developer.mozilla.org/ja/docs/Web/CSS/position

https://drafts.csswg.org/css-position
