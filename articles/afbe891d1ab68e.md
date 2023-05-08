---
title: "Nuxt3 の NuxtError.data に型をつける"
emoji: "🧱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nuxt3]
published: true
---

## 概要

Nuxt3 では `throw createError()` を使ってエラー情報を投げると `~/error.vue` の props でエラー情報を取得できます。
このエラー情報の型が `NuxtError` になります。

NuxtError は Nuxt3 が使用している H3 で定義されている [H3Error](https://www.jsdocs.io/package/h3#H3Error) の型の別名となっています。
H3Error はエラー情報を表すいくつかのパラメーターを持っていますが、そのなかの data というプロパティにエラーの追加情報を含めることができるようになっています。

この data は any として定義されていおり型チェックが効かないので、declare で拡張して型をつけたいと思います。

:::message alert
useError で返ってくる型は NuxtError ではないため、この方法だと型が付きません
:::

## 手順

### 1. /@types/nuxt-error.d.ts を作成する

※ ファイル名は任意

### 2. 型の定義を追加する

```ts:@types/nuxterror.d.ts
declare module "#app" {
  interface NuxtError {
    data: { // 設定したい任意の型
      additionalMessage: string;
    };
  }
}

export {};
```

:::message
※ インポート元が #app となる根拠のドキュメントが見つかりませんでしたが、 [issue](https://github.com/nuxt/nuxt/issues/12401) を参考に #app にしています。
:::

## 確認

```vue:app.vue
<script setup lang="ts">
const goToErrorPage = () => {
  throw createError({
    data: {
      additionalMessage: "test",
    },
    fatal: true, // クライアントサイドで error.vue を表示するために必要
  });

  // throw createError({
  //   data: {
  //     errorMessage: "additional message", // 型エラーが出る
  //   },
  //   fatal: true, // クライアントサイドで error.vue を表示するために必要
  // });
};
</script>

<template>
  <button @click="goToErrorPage">error</button>
</template>

```

```vue:error.vue
<script setup lang="ts">
const props = defineProps<{error: NuxtError}>()
</script>

<template>
  <div>
    <div>{{ props.error.data.additionalMessage }}</div>
    <!-- {{ props.error.data.errorMessage }} 型エラー -->
  </div>
</template>
```

## 補足

createError() でオブジェクトの直下にプロパティを設定しても取得できないので注意してください

```ts
// @types/nuxt-error.d.ts
declare module "#app" {
  interface NuxtError {
    additionalMessage: string;
  }
}

// app.vue
throw createError({
  additionalMessage: "test",
  fatal: true,
});

// error.vue
console.log(props.error.additionalMessage === undefined); // true
```
