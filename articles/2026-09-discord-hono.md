---
title: "Edgeで動くDiscord Botフレームワークを作ってる - discord-hono" # 記事のタイトル
emoji: "🔥" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["discord", "typescript", "hono"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

🤏 I have a Hono ~
🫱 I have a Discord ~
🫸🫷 **Discord Hono**

https://github.com/luisfun/discord-hono

## これは何？

- Discord Bot作りたいな～
- Cloudflare Workersに載せたいな～
- Honoっぽい書き方がいいな～

こういった願望から生まれた、エッジ向けのDiscord Botフレームワークです。

### もう少し詳しく

Discord Botを作るには、discord.jsやdiscrod.pyなど使うのが有名です。これらのフレームワークは、Gateway接続を維持するためのサーバーが必要でした。しかし弱点として、小さなBotでもサーバーの管理が必要になり、無料で作ろうとすると少し複雑な状況でした。

そこで、Discord Botをサーバレスなエッジ環境（特にCloudflare Workers）で稼働できないか調べたところ、周辺ツールはあるものの、簡単に作るためのフレームワークが調べた範囲ではありませんでした。

また、プロジェクトを立ち上げた当初、Honoの設計思想をとても良く思っており、私も同様の思想でフレームワークを作りたいと思いました。

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
