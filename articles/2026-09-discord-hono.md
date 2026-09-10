---
title: "Edgeで動くDiscord Botフレームワークを作ってる" # 記事のタイトル
emoji: "🔥" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["discord", "typescript", "hono"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

🤏 I have a Hono ~
🫱 I have a Discord ~
🫸🫷 **Discord Hono**

https://github.com/luisfun/discord-hono

## これは何か

- Discord Bot作りたいな～
- Cloudflare Workersに載せたいな～
- Honoっぽい書き方がいいな～

そんな思いから生まれたのが、このDiscord Honoです。

### もう少し詳しく

Discord Botを作るには、discord.jsやdiscord.pyなどのフレームワークを使うのが一般的です。これらはGateway接続を維持するためにサーバーを必要とします。そのため、小さなBotであってもサーバーの管理が必要になり、無料で運用しようとすると少し複雑でした。

そこで、Discord Botをサーバレスなエッジ環境、特にCloudflare Workersで動かせないか調べました。周辺ツールはいくつか見つかりましたが、手軽にBotを作るためのフレームワークは見当たりませんでした。

また、プロジェクトを立ち上げた当初から、Honoの設計思想に魅力を感じていました。そこで、同じような思想でフレームワークを作ることにしました。

## 設計思想

- サイズと処理の軽量化
- 実行時の依存関係なし
- TypescriptでDevXを提供

これらを基本的な設計思想とし、Honoライクなコーディング体験になるよう設計しました。

```ts
import { DiscordHono } from 'discord-hono'

const app = new DiscordHono()
  .command('hello', c => c.res('Hello, world!'))
  .command('about', c => c.res('discord-honoで動いています'))

export default app
```

コード例にある`'hello'`や`'about'`が、それぞれのコマンド名に当たります。
Honoらしい書き心地になっているのではないでしょうか。

## Discord Hono vs. discord.js

|  | Discord Hono | discord.js |
| --- | :-: | :-: |
| /command | ✅ | ✅ |
| VC接続 | ❌ | ✅ |
| チャット監視 | ❌ | ✅ |
| コスト | ✅ | ⚠️ |
| スケーリング | ✅ | ⚠️ |

### Botとしてできること

Discord Honoで利用できる機能には、いくつか制限があります。Gatewayへ接続しない仕組みのため、Gatewayに依存する機能は利用できません。ただし、1サーバー内のチャット監視であれば、1分ごとのcronとREST APIを組み合わせることで、疑似的に実現できます。

### コストについて

Cloudflare Workersにデプロイすれば、無料枠に収まるケースも多く、サーバー代をかけずに運用できます。また、コールドスタートもほぼゼロのため、スリープ対策として別のコードを実行する必要はありません。

### スケーリングについて

大規模なBotでの検証はできていないため、ここでの評価は理論上のものです。

Discord Interactions APIへのレスポンスにはレート制限がなく、Cloudflare Workersも実質的に無制限にスケールできます。レート制限で気にするのは、followupを含めるREST APIの利用や、Workerの背後に接続するデータベースやストレージです。

## 使い方やコード例

ドキュメントやコード例もいくつか作ってあるので、そちらを参考にしてください。

https://discord-hono.luis.fun/ja/guides/start/

https://github.com/luisfun/discord-hono-examples

リンクだけだと味気ないので、リンクのコード例をそのまま載せておきます。

### デプロイ用コード

https://github.com/luisfun/discord-hono-examples/blob/main/workerd-hello-world/src/index.ts

### コマンド登録用コード

https://github.com/luisfun/discord-hono-examples/blob/main/workerd-hello-world/src/register.ts

## 終わりに

小さなBotを無料で運用したい人は、ぜひDiscord Honoを検討してみてください。もし使ってみて気に入ったらリポジトリへスターを付けてくれると嬉しいです。

https://github.com/luisfun/discord-hono
