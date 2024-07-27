---
title: "Bunで環境変数を型安全に使う"
emoji: "🤓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bun", "typescript", "zod"]
published: true
published_at: 2024-07-29 09:00
---

環境変数が undefined の可能性をいちいち検証するのめんどくさいですよね
Bun で実行毎に環境変数を検証する方法を紹介します

## 手順

1. zod をインストール

```bash
bun add zod
```

2. env.ts 作成

```ts:env.ts
import { z } from 'zod'

const zVar = z.string().min(1)
const zEnv = z.object({
  // ここで必要な環境変数とそのスキーマを定義する
  DISCORD_CLIENT_ID: zVar,
  DISCORD_GUILD_ID: zVar,
  DISCORD_TOKEN: zVar,
})

const result = zEnv.safeParse(process.env)
if (!result.success) {
  throw `env type is invalid\n${result.error.errors.map((v) => `${v.message}: env.${v.path[0]}`).join('\n')}`
}

declare module 'bun' {
  interface Env extends z.infer<typeof zEnv> {}
}
```

3. bunfig.toml 作成

```diff toml:bunfig.toml
preload = ["./env.ts"]
```

4. 動作確認

```ts:main.ts
console.log("ababababa")
```

```bash:bash
bun run .
error: env type is invalid
Required: env.DISCORD_CLIENT_ID
Required: env.DISCORD_GUILD_ID
Required: env.DISCORD_TOKEN
```

## 解説

Bun は bunfig.toml というファイルで実行時の挙動をいろいろ設定できます
preload にパスを書いたファイルが、ファイルやスクリプトの実行時に先に走ります (便利！)
https://bun.sh/docs/runtime/bunfig#preload

env.ts では zod を使用して process.env がスキーマの期待する値か検証し、違ったらエラーを投げるようにしています
検証が通ったらそれらの環境変数は存在するので、実装時に undefined の場合を考えなくていいよう process.env の型を上書きすることで、string として扱えるようになります

## Node.js でもやりたい

パターン 1: dotenv みたいにファイルをインポートする (インポートしたファイルが走るまで検証されない)

```diff ts:env.ts
- declare module 'bun' {
-   interface Env extends z.infer<typeof zEnv> {}
- }
+ declare global {
+   namespace NodeJS {
+     interface ProcessEnv extends z.infer<typeof zEnv> {}
+   }
+ }
```

```ts:main.ts
import 'env'

console.log(env.DISCORD_TOKEN)
```

パターン 2: パース完了したものを process.env とは別で定義して使用する (パターン 1 では起こり得るインポートし忘れて検証できてないみたいな事故は起こらない)

```diff ts:env.ts
- declare module 'bun' {
-   interface Env extends z.infer<typeof zEnv> {}
- }
+ export const env = result.data
```

```ts:main.ts
import { env } from './env'

console.log(env.DISCORD_TOKEN)
```

Bun の快適さと比べるとやっぱり少ししょっぱいですね...

## おわりに

ちょっとした記事でさえ書くの大変だなーと感じます、たくさんの技術記事いつもお世話になっています
