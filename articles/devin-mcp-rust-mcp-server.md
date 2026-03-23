---
title: "Rust で Devin AI の MCP サーバーを作った"
emoji: "🦀"
type: "tech"
topics: ["Rust", "MCP", "DevinAI", "ClaudeCode", "AIエージェント"]
published: false
---

## 作ったもの

Claude Code や Cursor などの AI コーディングツールから、Devin AI にタスクを委譲できる MCP サーバーを Rust で作った。

https://github.com/mjinno09/devin-mcp

MCP（Model Context Protocol）は、AI ツールに外部機能を追加するための標準プロトコル。このサーバーを入れると、Claude Code の会話中に「この作業は Devin にやらせて」と言うだけで、Devin のセッションが立ち上がり、結果は GitHub PR として返ってくる。

## モチベーション

### MCP サーバーを作ってみたかった

MCP（Model Context Protocol）が盛り上がっている中で、単純に自分で MCP サーバーを作ってみたかった。プロトコルの仕様を読むだけでなく、実際に動くものを作ることで理解が深まると思った。

### Claude Code + Devin のワークフローに必要だった

普段の開発では、Claude Code で GitHub issue を作成し、その issue を Devin に自律的に実装させるというワークフローで作業している。Claude Code でコードを読みながら issue を整理し、実装は Devin に任せて PR を待つ、という流れだ。

ただ、毎回ブラウザで Devin のダッシュボードを開いてプロンプトを書いて...というコンテキストスイッチが地味に面倒だった。Claude Code の会話の中からそのまま Devin にタスクを投げられれば、ターミナルから離れずに済む。

MCP サーバーとして実装すれば、Claude Code・Cursor・その他 MCP 対応クライアントのどれからでも同じように使える。

## 提供するツール

devin-mcp は 4 つの MCP ツールを公開する。

| ツール | 用途 |
|---|---|
| `create_session` | Devin にタスクを投げる。セッション URL を返して即完了（fire-and-forget） |
| `get_session` | セッションの状態・メッセージ・PR 情報を取得 |
| `list_sessions` | 直近のセッション一覧を取得 |
| `send_message` | 実行中のセッションにフォローアップ指示を送信 |

`create_session` にはオプションで `title`、`tags`、`max_acu_limit`（計算リソース上限）、`playbook_id`、`snapshot_id` などを指定できる。

## アーキテクチャ

```
MCP Client (Claude Code / Cursor)
    ↓  stdio, JSON-RPC 2.0
devin-mcp Server (Rust binary)
    ↓  HTTPS, Bearer Token
Devin API (api.devin.ai/v1)
    → GitHub PR
```

設計方針:

- **ステートレス** — セッション情報をサーバー側に保持しない。毎回 Devin API に問い合わせる
- **Fire-and-forget** — `create_session` は URL を返した時点で完了。ポーリングしない
- **stdout は JSON-RPC 専用** — ログは全て stderr に出力。stdout に余計な出力を混ぜると MCP プロトコルが壊れる

## 技術スタック

| 項目 | 選定 |
|---|---|
| 言語 | Rust (2021 edition) |
| MCP SDK | rmcp v1 |
| HTTP クライアント | reqwest 0.13 |
| 非同期ランタイム | tokio |
| シリアライズ | serde + serde_json |
| JSON Schema 生成 | schemars |
| ロギング | tracing |

### なぜ Rust か

MCP サーバーは stdio で起動されるプロセスなので、起動速度が体感に直結する。Rust ならコールドスタートが速い。また、シングルバイナリにコンパイルされるため、`brew install` 一発で入り、ランタイムの依存がない。Node.js や Python のように環境を整える必要がなく、どのマシンでも同じバイナリをそのまま動かせる。

### なぜ rmcp か

MCP SDK には rmcp を選んだ。Rust 向けの MCP SDK としてはほぼ唯一のまともな選択肢で、`#[tool]` マクロによる宣言的なツール定義ができる。

## 実装のポイント

### `#[tool]` マクロでツールを宣言的に定義

rmcp の `#[tool]` マクロを使うと、関数にアトリビュートを付けるだけで MCP ツールとして公開できる。

```rust
#[tool(description = "Create a new Devin session")]
async fn create_session(
    &self,
    #[tool(param)]
    params: CreateSessionParams,
) -> Result<CallToolResult, McpError> {
    // ...
}
```

### doc comment が LLM へのパラメータ説明になる

パラメータ構造体のフィールドに書いた doc comment が、schemars 経由で JSON Schema の `description` に変換される。これが MCP クライアント経由で LLM に渡り、パラメータの意味を理解する手がかりになる。

```rust
#[derive(Deserialize, JsonSchema)]
pub struct CreateSessionParams {
    /// タスクの説明。Devin に実行させたい作業内容を具体的に記述する
    pub prompt: String,

    /// セッションのタイトル（省略時は自動生成）
    pub title: Option<String>,
}
```

### セッション ID のバリデーション

Devin API のエンドポイントにセッション ID を含めるため、パストラバーサル対策としてバリデーションを入れた。英数字・ハイフン・アンダースコアのみ許可する。

```rust
fn validate_session_id(id: &str) -> Result<()> {
    if id.chars().all(|c| c.is_alphanumeric() || c == '-' || c == '_') {
        Ok(())
    } else {
        Err(anyhow!("Invalid session ID"))
    }
}
```

### ログは stderr に限定

MCP サーバーは stdin/stdout で JSON-RPC 2.0 通信を行うため、`println!` は厳禁。全てのログ出力を `tracing` 経由で stderr に向ける。

```rust
tracing_subscriber::fmt()
    .with_writer(std::io::stderr)
    .init();
```

## セットアップ

### インストール

```sh
brew install mjinno09/tap/devin-mcp
```

### Claude Code に追加

```sh
claude mcp add --transport stdio --scope user \
  --env DEVIN_API_KEY="${DEVIN_API_KEY}" \
  devin-mcp -- devin-mcp
```

### Cursor に追加

```json:.cursor/mcp.json
{
  "mcpServers": {
    "devin-mcp": {
      "command": "devin-mcp",
      "env": {
        "DEVIN_API_KEY": "your_api_key_here"
      }
    }
  }
}
```

`DEVIN_API_KEY` は [Devin Settings](https://app.devin.ai/settings) から取得する。

## 使い方

Claude Code で以下のように指示するだけで Devin にタスクが委譲される。

```
このリポジトリの README を日本語に翻訳して PR 作って。Devin に任せて。
```

Claude Code が `create_session` ツールを呼び出し、Devin のセッション URL が返ってくる。あとは Devin が非同期で作業し、完了すると GitHub PR が作られる。

進捗を確認したければ:

```
さっき作った Devin セッションの状態を確認して
```

追加の指示を送ることもできる:

```
Devin に「テストも追加して」と伝えて
```

## まとめ

MCP サーバーとして実装することで、特定のクライアントに依存せず、Claude Code・Cursor・その他 MCP 対応ツールのどれからでも Devin にタスクを委譲できるようになった。

Rust + rmcp の組み合わせは、マクロベースの宣言的な記述で MCP サーバーを簡潔に書ける。型安全性と doc comment → JSON Schema の自動変換により、LLM にとっても人間にとっても分かりやすいインターフェースが作れる。

リポジトリ: https://github.com/mjinno09/devin-mcp
