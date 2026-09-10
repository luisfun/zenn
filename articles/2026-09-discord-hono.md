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

## discord-honoが解決する世界

### コールドスタートほぼゼロのHTTP Bot

Gateway接続型のBotでは、プロセスの起動後に接続を維持する必要があります。HTTP型のBotでは、DiscordからのInteractionが実行のきっかけです。

Cloudflare Workersはリクエスト単位でコードを実行するため、Bot用サーバーを自分で起動し続ける必要がありません。エッジ上のWorkersランタイムで処理されるので、従来のサーバーレス環境で気になりやすいコールドスタートをほとんど意識せずに済みます。

### サーバー代0円で始める

Cloudflare Workersの無料枠に収まる規模であれば、サーバーをレンタルせずにBotを公開できます。必要になるのは、主に次のものです。

- Discordアプリケーション
- Cloudflareアカウント
- コードを置くリポジトリ

大量のリクエスト、常時大量のログ保存、外部データベースなどを使う場合は、無料枠や各サービスの料金を確認してください。それでも、試作や個人用Botの最初の一歩として、固定費を抑えやすい構成です。

### 使いどころを整理する

discord-honoとdiscord.jsは、どちらが優れているというより得意な接続方式が違います。

| 目的 | 向いている選択肢 |
| --- | --- |
| Slash CommandやボタンへのHTTP応答 | discord-hono |
| Gatewayイベントを常時受信する | discord.js |
| ボイスチャンネルへの接続や音声制御 | discord.js |
| サーバー上のメッセージを監視し続ける | discord.js |
| Cloudflare Workersへ小さなBotをデプロイする | discord-hono |

メッセージのイベントを常時監視したい、音声を再生したい、Discordの豊富なイベントを購読したい、といった場合はGateway接続が必要です。その場合はdiscord.jsなどのGateway対応ライブラリが適しています。

反対に、コマンドを受けて短い処理をし、Interactionへ応答するBotならdiscord-honoが候補になります。データをD1やKVに保存する処理も、WorkersのBindingと組み合わせて実装できます。

## ハンズオン: Hello World Bot

ここでは、`/hello` に応答するBotをCloudflare Workersへデプロイします。

### 1. プロジェクトを作成する

Cloudflare Workersのプロジェクトを作成し、ライブラリをインストールします。

```sh
npm create cloudflare@latest discord-hono-hello
cd discord-hono-hello
npm i discord-hono
npm i -D discord-api-types
```

TypeScriptを使う構成を選択してください。

### 2. Workerを書く

`src/index.ts` を次の内容にします。

```ts
import { DiscordHono } from 'discord-hono'

const app = new DiscordHono()
	.command('hello', c => c.res('Hello, World!'))

export default app
```

`DiscordHono` は `fetch` を持つWorkerとして動作します。GETリクエストには稼働確認用のレスポンスを返し、POSTリクエストでは署名検証後にInteractionを処理します。

### 3. Discordのコマンドを登録する

コマンドの登録は、Worker本体とは別のスクリプトで一度実行します。`src/register.ts` を作成します。

```ts
import { makeSlashCommand, register } from 'discord-hono'

const commands = [
	makeSlashCommand('hello', 'Hello, World!'),
]

register(
	commands,
	process.env.DISCORD_APPLICATION_ID,
	process.env.DISCORD_TOKEN,
)
```

`package.json` に登録用スクリプトを追加します。

```json
{
	"type": "module",
	"scripts": {
		"register": "tsc && node --env-file=.env dist/register.js",
		"deploy": "wrangler deploy"
	}
}
```

### 4. Discordアプリを設定する

[Discord Developer Portal](https://discord.com/developers/applications)でアプリケーションを作成し、次の値を取得します。

- Application ID
- Public Key
- Bot Token

ローカル登録用の `.env` に値を設定します。

```dotenv
DISCORD_APPLICATION_ID=アプリケーションID
DISCORD_PUBLIC_KEY=公開鍵
DISCORD_TOKEN=Botトークン
```

`.env` はリポジトリへコミットしません。`npm run register` で `/hello` コマンドを登録できます。テスト用のGuild IDを指定して登録すると、反映を確認しやすくなります。

### 5. デプロイしてEndpointを設定する

```sh
npx wrangler secret put DISCORD_APPLICATION_ID
npx wrangler secret put DISCORD_PUBLIC_KEY
npx wrangler secret put DISCORD_TOKEN
npm run deploy
```

`wrangler secret put` はそれぞれのコマンドで値を入力します。デプロイ後に表示されたWorkerのURLを、Discord Developer Portalの **Interactions Endpoint URL** に設定してください。

設定が完了すると、Discordから送られた `/hello` にWorkerが応答します。Discord側の検証リクエストが成功しない場合は、Endpoint URL、`DISCORD_PUBLIC_KEY`、そしてWorkerに設定したSecretを確認します。

## 終わりに

discord-honoは、Discord Botを常時接続するプロセスではなく、HTTPリクエストを処理するWorkerとして捉え直すためのライブラリです。

Honoに影響を受けたAPIとTypeScriptによる型の担保を用意しました。Workers標準のWeb Crypto APIで署名も検証します。DiscordのInteractionに集中してコードを書ける設計です。

すべてのBotをHTTP型に置き換えられるわけではありません。音声やGatewayイベントが必要ならdiscord.jsが自然です。一方、コマンドやコンポーネントへの応答が中心なら、エッジ環境で小さく始められるdiscord-honoを選択肢にできます。

- [discord-honoのGitHubリポジトリ](https://github.com/luisfun/discord-hono)
- [公式ドキュメント](https://discord-hono.luis.fun/)
- [サンプルリポジトリ](https://github.com/luisfun/discord-hono-examples)
