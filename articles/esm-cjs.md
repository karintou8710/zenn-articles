---
title: CJS/ESM のdefault ~TypeScriptを添えて~
type: tech
topics: []
emoji: 🔖
published: false
---
zenn-cli に wysiwyg エディターを追加しようと作業をしていたのですが、そこで CJS である `zenn-markdown-html` をブラウザでも実行可能にしようと、ESM に書き換えていました。

しかし、ブラウザで実行すると期待通りだったのですが、サーバー側の CJS で実行すると import の一部がなぜか `{default: fn}` の形式になり、デフォルトエクスポートが出来ておらずエラーになりました。

調べてみると、`zenn-markdown-html` の依存先である CJS の [@steelydylan/markdown-it-imsize](https://www.npmjs.com/package/@steelydylan/markdown-it-imsize) が関連しているっぽいです。

本記事ではこの現象が起きた理由を、CJS, ESM, TypeScript や esbuild といったトランスパイラの観点から調べます。

## CJS と ESM

「CJS と ESM の違い？ あ〜わかるよ。require か import の違いでしょ？？」

ってぐらいの知識しかなかったので、基本から振り返るために JavaScript と TypeScript のモジュールシステムについて調べてみます。

ここからは、以下のフォルダ構造を前提に話します。pnpm のモノレポです。

```:フォルダ構造
├── package.json
├── packages
│   ├── cjs
│   │   ├── main.js
│   │   ├── mod-default.js
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

**ESM**

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

ESM では０個以上の名前付きエクスポートで実現します。デフォルトエクスポートは、default という予約文字の名前付きエクスポートです。

---

**CJS**

```js
// cjs/mod-default.js
module.exports = "default export";

// cjs/main.js
const modDefault = require("./mod-default");
console.log(modDefault); // default export
```

```js
// cjs/mod-name.js
module.exports.name = "named export";
module.exports.default = "fake default export";

// cjs/main.js
const modName = require("./mod-name");
console.log(modName.default); // "fake default export
console.log(modName.name); // "named export"
```

CJS では、単一のエクスポートを `module.exports` に代入することで実現します。

`module.exports` が CJS におけるデフォルトエクスポートになっており、`exports.name`で名前付きエクスポートになります。

`module.exports.default` といった表現を目にしますが、これは名前付きエクスポートなので注意してください。後ほど出てくるトランスパイラの力を借りて、ようやくデフォルトエクスポートとして認識されます。

:::details ESM のランタイム
ブラウザ用だから ESM は Node では実行できないんでしょ？と思ってましたが、意外にも Node で実行可能です。

ESM 対応は [Node v14 で stable](https://nodejs.medium.com/node-js-version-14-available-now-8170d384567e) になったみたいです。（導入されたのは v12）

逆に、CJS は現在でも主要のブラウザ上で実行する事はできません。
:::

### 相互インポート

ESM と CJS を相互にインポートする方法を考えてみます。

#### ESM → CJS

これは v12 の時点では既に可能だったみたいです。`module.exports` が、ESM のデフォルトエクスポートとして認識されます。

ここら辺の挙動は [Node のドキュメント](https://nodejs.org/api/esm.html#interoperability-with-commonjs)に載ってました。

```js:esm/main.js
import mod from 'cjs/mod-default';
console.log(mod) // "default export"
```

まず `module.exports`を読み込んでみると、意図した通りの挙動になります。

これは CJS の `module.exports` を、 ESM の default の名前付きインポートとして内部的に変換されることで、ESM でのデフォルトエクスポートが可能になっています。

```js:esm/main.js
import ModName from 'cjs/mod-name' // { name: ..., default: ...}
// SomeDefault === ModName になり、名前付きインポートではdefaultを取得できない
import { default as SomeDefault } from 'cjs/mod-name'

console.log(ModName.name) // "Named Export"
console.log(ModName.default) // "Fake Default Export"
console.log(SomeDefault === ModName) // true
```

CJS の名前付きエクスポートは ESM でデフォルトインポートができます。

また、名前付きインポートも通常可能ですが、`default` は例外です。ESM から default を名前付きインポートすると、デフォルトインポートしたものオブジェクトと同じものが取得されます。

それなので default という名前付きエクスポートを取得するためには、一度全体をデフォルトインポートしてから、default のプロパティを指定します。

#### CJS → ESM

こちらは [Node v23](https://nodejs.org/en/blog/release/v23.0.0/) で stable になったみたいです。≈

[ドキュメント](https://nodejs.org/api/modules.html#loading-ecmascript-modules-using-require)によると、デフォルトエクスポートは `module.default` の名前付きのエクスポートになります。

なので、`const { default } = require('./esm')` のように、一度展開する必要があります。

```js:cjs/main.js
const mod = require("esm")
mod.default() // "Hello, ESM"
console.log(mod.namedExport) // "named export"
```

実際に試してみると、default という名前付きインポートになってました。

### ESM と CJS におけるデフォルト変換の問題点

例えば、ESM で以下の構造を考えてみます。

```js
const obj = { a: 1 };

export const a = 2;
export default obj;
```

この時、 `export default obj` を `const obj = require()` で取得するのが自然だと思いますが、CJS は 単一のオブジェクトでモジュールを管理しているため、`変数 a` が競合してしまいます。

そこで Node の CJS → ESM で確認した通り、以下の様になります。

```js
module.exports = {
  default: { a: 1 },
  a: 2,
};
```

ESM の デフォルトエクスポートは、default の名前付きエクスポートに変換されます。しかし、この方法だと require する時に `obj.default` と展開する必要性があったりと、直感と異なります。

逆に ESM → CJS では、`module.exports` をデフォルトエクスポートとするのが自然なため、`default` の名前付きエクスポートは挙動に影響を与えません。

ここら辺は TypeScript や Babel の力を借りて自然に書けるのですが、Node 単体ではこの問題に対処できません。

## TypeScript が絡むと

ややこしい話でしたが、TypeScript が絡むと更に複雑になります。

ESM → CJS へのトランスパイルの過程で、

`import mod from "cjs"`を、ESMの意味論で `require("cjs").default`に変換する。

import \* as mod from "cjs" を、ESM の意味論で rewuire("cjs") に変換する。

この挙動をいい感じにしたのが esModuleInteropフラグになる。

\__esModuleフラグが設定されているか否かで、{ default: fn } でラップするか決める。

\__esModuleフラグはESM → CJSにトランスパイルされる時に付与される。

require した要素を全て { default: fn} の形式にしてから、`require("cjs").default`とすることでデフォルトエクスポートを可能にしている。

### zenn-markdown-html では何が起こっていたのか

 

### 参考

https://zenn.dev/qnighy/articles/6267716578c76d
