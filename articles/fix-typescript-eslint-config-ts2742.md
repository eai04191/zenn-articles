---
title: "typescript-eslintのconfigでTS2742エラーが出るときの対処"
emoji: "🔧"
type: "tech"
topics: [typescript, eslint, pnpm]
publication_name: "ncdc"
published: true
---

> 'default' の推論された型には、'.pnpm/@typescript-eslint+utils@8.37.0_eslint@9.31.0_typescript@5.8.3/node_modules/@typescript-eslint/utils/ts-eslint' への参照なしで名前を付けることはできません。これは、移植性がない可能性があります。型の注釈が必要です。ts(2742)

何を言っているんだという感じのエラーですが、推論された型が複雑すぎてヤバいよ！ということらしいです。重要なのは「型の注釈が必要です。」の部分です。'default'の型はコレ！と言えればよいようです。

なので型をつけましょう。現状こうなっているはずです。

<!-- prettier-ignore -->
```js
export default tseslint.config(
    // ... 
);
```

アノテーションをつけられるようにオブジェクトにします。

<!-- prettier-ignore -->
```js
const config = tseslint.config(
    // ...
);

export default config;
```

アノテーションをつけます。

<!-- prettier-ignore -->
```js
/** @type {import('typescript-eslint').Config} */
const config = tseslint.config(
    // ...
);
export default config;
```

これで文句を言われなくなったはずです。
