---
title: "Edgeで動くDiscord Botライブラリを作った - discord-hono" # 記事のタイトル
emoji: "🔥" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["discord", "typescript", "hono"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

Discord Botを作るとき、まず名前が挙がるライブラリのひとつが [discord.js](https://discord.js.org/) です。豊富な機能と大きなエコシステムがあり、Gatewayに接続してイベントを受け取るBotにはとても心強い選択肢です。

一方で、スラッシュコマンドやボタンへの応答が中心のBotなら、常時接続を維持しなくても実現できます。DiscordのInteractionsをHTTPリクエストとして受け取り、処理してレスポンスを返すだけです。

この形に合わせて、Cloudflare Workers向けのDiscord Botライブラリ [discord-hono](https://github.com/luisfun/discord-hono) を作りました。この記事では、作った背景、設計上の工夫、そして簡単なBotを動かすまでを紹介します。

## なぜ作ったか

### 開発に至った背景

これまでDiscord Botを作るときは、Node.jsのプロセスを起動し、Discord Gatewayへ接続する構成が自然でした。しかし、コマンドに応答するだけのBotでは、実際に必要なのは次のような処理です。

1. DiscordからHTTPリクエストを受け取る
2. リクエストがDiscordから来たものか検証する
3. コマンドに対応する処理を実行する
4. Interactionへのレスポンスを返す

この用途に、常時起動するサーバーやGateway接続は必須ではありません。Cloudflare Workersのようなエッジ環境なら、リクエストが来たときだけコードを実行できます。

ただし、DiscordのInteractionを自分でルーティングし、レスポンスのJSONや署名検証まで実装すると、Bot本体より周辺処理が目立ちます。Honoのようにリクエストを受けてハンドラを書く感覚で、Discord Botを作れるライブラリが欲しくなりました。こうしてdiscord-honoの開発を始めました。

### エッジ環境でBotを動かす利点

Cloudflare WorkersにデプロイしたBotは、特定のサーバーを自分で管理する必要がありません。リクエストはCloudflareのネットワーク上で処理され、アプリケーションはHTTPの入口だけを持ちます。

この構成には、次のような利点があります。

- サーバーの起動、監視、OSの更新が不要
- Bot用プロセスを常時起動しておく必要がない
- 世界中のエッジからリクエストを受けられる
- 小規模なBotなら、Cloudflare Workersの無料枠で運用できる

もちろん無料枠にはリクエスト数や実行時間などの制限があります。また、外部APIやデータベースを使えば、そのサービスの料金も発生します。「サーバー代0円」は、無料枠の範囲に収まる小規模なBotを想定した表現です。

## 設計思想と技術的な工夫

### Honoライクな書き方

discord-honoの中心は `DiscordHono` クラスです。コマンド名とハンドラをチェーンで登録できます。

```ts
import { DiscordHono } from 'discord-hono'

const app = new DiscordHono()
	.command('hello', c => c.res('Hello, World!'))
	.command('about', c => c.res('discord-honoで動いています'))

export default app
```

Cloudflare Workersのエントリポイントとして `app` をそのままexportできます。ハンドラに渡されるコンテキストからは、Interactionのデータや環境変数、Discord REST APIを呼び出すための機能にアクセスできます。

ボタンやモーダルにも同じ考え方を適用できます。

```ts
import { DiscordHono, makeActionRow, makeButton } from 'discord-hono'

const app = new DiscordHono()
	.command('hello', c =>
		c.res({
			content: 'ボタンを押してください',
			components: [makeActionRow([makeButton('delete', ['削除', 'Delete'])])],
		}),
	)
	.component('delete', c => c.update().res('削除しました'))

export default app
```

### 型を担保する

DiscordのInteractionやコンポーネントには、種類ごとに異なるデータ構造があります。discord-honoでは、Discordが提供する `discord-api-types` を利用し、コマンド、コンポーネント、オートコンプリート、モーダルなどのハンドラに型を付けています。

たとえばスラッシュコマンドのオプションは、Builderで定義した内容に沿って扱えます。

```ts
import { DiscordHono, makeSlashCommand, makeStringOption } from 'discord-hono'

const command = makeSlashCommand(
	'hello',
	'挨拶する',
	makeStringOption('name', '名前', { required: true }),
)

const app = new DiscordHono()
	.command(command.name, c => {
		const name = c.var.options.name
		return c.res(`Hello, ${name}!`)
	})

export default app
```

実際のBuilderの引数やコンテキストのプロパティはバージョンによって変わるため、利用時には[公式ドキュメント](https://discord-hono.luis.fun/)とエディタの型情報を確認してください。ライブラリの目的は、DiscordのJSONを直接扱う箇所をできるだけ減らすことです。

### Ed25519署名の検証

DiscordのInteractions Endpointには、次のヘッダーが付いてきます。

- `x-signature-ed25519`: 署名
- `x-signature-timestamp`: 署名に使われたタイムスタンプ

検証対象は、タイムスタンプとリクエスト本文を連結したバイト列です。アプリケーションの公開鍵を使ってEd25519署名を検証し、検証に失敗したリクエストはハンドラへ渡しません。

discord-honoでは、この処理をCloudflare Workersで利用できるWeb Crypto APIの `crypto.subtle` で行っています。

```ts
const verified = await crypto.subtle.verify(
	{ name: 'Ed25519' },
	await crypto.subtle.importKey(
		'raw',
		publicKeyBytes,
		{ name: 'Ed25519' },
		false,
		['verify'],
	),
	signatureBytes,
	new TextEncoder().encode(timestamp + body),
)
```

Node.js用の暗号ライブラリを持ち込まず、Workersの標準APIだけで完結するのがポイントです。本文は検証前に `request.text()` で読み、その同じ本文を検証とJSONパースに使います。署名検証のために本文を書き換えないことも重要です。

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
