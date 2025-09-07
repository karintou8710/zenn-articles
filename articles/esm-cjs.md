---
title: CJS, ESM のモジュール解決にトランスパイラを添えて
type: tech
topics: []
emoji: 🔖
published: false
---
zenn-cli に wysiwyg エディターを追加しようと作業をしていたのですが、そこで CJS である `zenn-markdown-html` をブラウザでも実行可能にしようと、ESM に書き換えていました。

しかし、ブラウザで実行すると期待通りだったのですが、サーバー側の CJS で実行すると import がなぜか `{default: fn}` の形式になり、エラーになりました。

調べてみると、`zenn-markdown-html` の依存先である CJS の [@steelydylan/markdown-it-imsize](https://www.npmjs.com/package/@steelydylan/markdown-it-imsize) が関連しているっぽいです。

本記事ではこの現象が起きた理由を、CJS, ESM, トランスパイラーの観点から調べていきます。

## CJS と ESM

「CJS と ESM の違い？ あ〜わかるよ。require か import の違いでしょ？？」

ってぐらいの知識しかなかったので、基本から振り返るために JavaScript と TypeScript のモジュールシステムについて調べてみます。

ESModule(ESM) はブラウザで採用されている [ECMAScript の仕様](https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#sec-modules)にあるモジュールシステムです。以下のように `import` `export` を書けます。

CommonJS(CJS) はサーバー (Node)で採用されている仕様で、その中に `require` と `import` によるモジュールシステムがあります。

---

ここからは、以下のフォルダ構造を前提に話します。pnpm のモノレポです。

```:フォルダ構造
├── package.json
├── packages
│   ├── cjs
│   │   ├── main.js
│   │   ├── mod-default.js
│   │   ├── mod-mix.js
│   │   ├── mod-name.js
│   │   └── package.json
│   └── esm
│       ├── main.js
│       ├── mod.js
│       └── package.json
├── pnpm-lock.yaml
└── pnpm-workspace.yaml
```

### 基本的な文法

**ESM は以下です。**

```js:esm/main.js
import mod, {namedExport} from './mod.js';

mod(); // "Hello, ESM"
console.log(namedExport); // "named export"
```

```js:esm/mod.js
export default function hello() {
    console.log("Hello, ESM")
}

export const namedExport = "named export";
```

`main.js` を `type="module"`のスクリプトとしてブラウザで読み込むと、現在のパスから `sub.js` を探し、`hello`を読み込んでくれます。特徴的なのは拡張子が必須なことです。

デフォルトエクスポートと名前付きエクスポートはよくみるやつです。

**CJS は主に 3 つの記法がありました。**

```js:cjs/mod-default.js
module.exports = "default export"
```

```js:cjs/mod-name.js
module.exports.name = "Named Export"
module.exports.default = "Fake Default Export"
```

```js:cjs/mod-mix.js
const func = () => {
  console.log("Hello, CJS")
}
func.namedExport = "Fake Named Export"
module.exports = func
```

```js:cjs/sub.js
const modDefault = require('./mod-default');
console.log(modDefault) // default export

const modName = require('./mod-name');
console.log(modName.default) // "Fake Default Export
console.log(modName.name) // "Named Export"

const modMix = require('./mod-mix');
modMix() // "Hello, CJS"
console.log(modMix.namedExport) // "Fake Named Export"
```

`module.exports` が ESM におけるデフォルトエクスポートになっており、`exports.name`で名前付きエクスポートになります。拡張子はいい感じに解決してくれます。

若干特殊ですがオブジェクトにプロパティを加える形で、名前付きエクスポートを同時にすることも可能です。

`exports.default` といった表現を目にしますが、これは名前付きエクスポートなので注意してください。後ほど出てくるトランスパイラの力を借りて、ようやくデフォルトエクスポートとして認識されます。

### ランタイムの制約

ブラウザ用だから ESM は Node では実行できないんでしょ？と思ってましたが、意外にも Node で実行可能です。

実は ESM 対応は、Node v14 で stable になったみたいです。（導入されたのは v12）

https://nodejs.medium.com/node-js-version-14-available-now-8170d384567e

逆に、CJS は現在でも主要のブラウザ上で実行する事はできません。

### 相互インポート

上記の話はあくまでどちらかのモジュールで統一されていた場合なので、実際は相互にインポートすることになることがあります。それらを確認していきます。

パッケージのインポートのため、`package.json` に `export` を設定します。

#### ESM → CJS

これは v12 の時点では既に可能だったみたいです。`module.exports` が、ESM のデフォルトエクスポートとして認識されます。

```js:esm/main.js
import mod from 'cjs/mod-default';
console.log(mod) // default export
```

まずデフォルトエクスポート単体を読み込んでみると、意図した通りの挙動になります。

```js:esm/main.js
import ModName from 'cjs/mod-name' // { name: ..., default: ...}
// import { default } from 'cjs/mod-name' default を名前付きインポートすることはできない
import { default as SomeDefault } from 'cjs/mod-name' // SomeDefault === ModName になり、ModName.defaultは取得不可


console.log(ModName.name) // "Named Export"
console.log(ModName.default) // "Fake Default Export"
console.log(SomeDefault === ModName) // true
```

名前付きエクスポートは、若干雲行きが怪しいです。

CJS の名前付きエクスポートは ESM でデフォルトインポートができます。この場合、意図した通りのオブジェクトになります。

また、名前付きインポートも通常可能ですが、`default` は例外です。まず、default はキーワード登録されているため、そのままインポートできないです。
また名前を変えたとしても、ESM から default を名前付きインポートすると、デフォルトインポートしたものオブジェクトと同じものが取得されます。

これは ECMAScript の仕様に記載されているみたいです。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/import

```js:esm/main.js
import modMix from 'cjs/mod-mix'
// import { namedExport } from 'cjs/mod-mix' 関数のプロパティに追加しただけなため、名前付きインポートすることはできない
modMix() // "Hello, CJS"
console.log(modMix.namedExport) // "Fake Named Export"
```

最後に混合した場合だと、名前付きインポートができなくなります。
この挙動はよく考えるとそうで、デフォルトエクスポートの関数にプロパティを加えただけだからです。

#### CJS → ESM

こちらは [Node v23](https://nodejs.org/en/blog/release/v23.0.0/) で stable になったみたいです。

https://nodejs.org/api/modules.html#loading-ecmascript-modules-using-require

ドキュメントによると、デフォルトエクスポートは `module.default` という名前付きのエクスポートになるみたいです。

なので、`const { default: hello } = require('./sub-esm.mjs');` のように、一度展開する必要があります。

```js:cjs/main.js
const mod = require("esm")
mod.default() // "Hello, ESM"
console.log(mod.namedExport) // "Named Export"
```

実際に試してみると、default という名前付きインポートになってました。

## TypeScript が絡むと

ここまでもややこしい話でしたが、TypeScript が絡むと更に複雑になります。
もう既に考えたくないですね。。。

TypeScript のモジュール仕様は以下にありました。

https://www.typescriptlang.org/docs/handbook/modules/introduction.html

### TypeScript における import
