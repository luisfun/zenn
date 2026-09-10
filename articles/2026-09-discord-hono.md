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

Discord Honoができることは制限されています。これは、Gateway接続ができないため、それに関わる機能を利用できないからです。
ただし、この表の中で、1サーバーのチャット監視であれば疑似的にできます。（1分毎のcronとREST APIの組み合わせで可能）

### コストについて

Cloudflare Workersへのデプロイならば、無料枠で済むことが多く、サーバー代を0円にすることが可能です。
コールドスタートがほぼゼロなので、スリープ対策用にヘルスチェックを定期的に叩く必要もありません。

### スケーリングについて

開発過程で大規模Botのテストをできていないため、推測評価です。

理論上ではありますが、Discord Interactions APIはレート制限がなく、Cloudflare Workersも実質無限スケーリングです。スケーリングのボトルネックになるのは、workerの後ろに接続するデータベースやストレージです。

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
