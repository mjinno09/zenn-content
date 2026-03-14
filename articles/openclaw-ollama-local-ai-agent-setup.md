---
title: "OpenClaw + Ollama でローカル AI エージェントを構築した記録"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "Ollama", "Docker", "LLM", "AIエージェント"]
published: false
---

# OpenClaw + Ollama でローカル AI エージェントを構築した記録

MacBook（Apple Silicon / 128GB RAM）上に、OpenClaw と Ollama を使ったローカル AI エージェント環境を構築した。クラウド API を使わず、完全ローカルで動く AI エージェントを目指した一日の記録。

## やりたかったこと

- **完全ローカルで動く AI エージェント**を構築する
- OpenClaw（OSS の AI エージェント Gateway）を Docker で動かす
- LLM は Ollama 経由でローカルモデルを使う（API課金なし）
- エージェントの人格・記憶・ルールを workspace ファイルで管理し、GitHub Private Repo で永続化する

## やったこと

### 1. GitHub Private Repo の作成

```bash
gh repo create openclaw-agent --private
```

エージェントの workspace（人格定義、記憶ログ等）を Git 管理するためのリポジトリを作成。

### 2. Docker 環境の構築

OpenClaw を Docker コンテナとして動かす構成を作成。

**Dockerfile の要点:**
- `node:24-slim` ベース
- npm 依存の git+ssh 問題を `insteadOf` で HTTPS にリライト
- 非 root ユーザーで実行
- `openclaw.json` を `COPY` でイメージにベイク

**docker-compose.yml の要点:**
- workspace をホストとマウント共有（エージェントの記憶が Mac 側に反映）
- シークレットは `.env` で管理（gitignore）

### 3. Ollama のローカル実行

当初は Ollama も Docker 内で動かしていたが、メモリ問題で Mac ローカルに移行。

```bash
brew install ollama
brew services start ollama
ollama pull qwen3.5:9b
```

Docker コンテナからは `host.docker.internal:11434` で接続。

### 4. モデル選定の旅

```
Gemma 3 12B → ツール非対応で断念
  ↓
Qwen 3.5 9B → ツール対応、ローカル Ollama で安定動作
```

### 5. Workspace ファイルの設計

```
workspace/
├── SOUL.md         # エージェントの人格・哲学
├── IDENTITY.md     # 名前・キャラクター設定
├── AGENTS.md       # セッション起動ルール・SOP
├── USER.md         # ユーザー情報
├── MEMORY.md       # 長期記憶サマリ
├── TOOLS.md        # スキル・ツール設定
├── HEARTBEAT.md    # 定期タスク
└── memory/         # 日次記憶ログ（エージェントが書き込む）
    └── 2026-03-14.md
```

### 最終アーキテクチャ

```
ブラウザ ──WS──▶ OpenClaw Gateway (Docker :18789) ──HTTP──▶ Ollama (Mac :11434)
                       │                                        │
                  /app/workspace ◀──マウント──▶ ~/openclaw-agent/workspace/
                                                    │
                                               git push → GitHub
```

| コンポーネント | 実行場所 | 理由 |
|---|---|---|
| OpenClaw Gateway | Docker | 隔離・再現性 |
| Ollama + Qwen 3.5 9B | Mac ローカル | Apple Silicon 統合メモリ活用 |
| Workspace | ホスト（Git管理） | 記憶の永続化・バージョン管理 |

## トラブルシューティング（全16件）

セットアップ中に遭遇した問題を時系列で記録する。

### Docker ビルド編

**1. `node:24-slim` に git がない**
OpenClaw の npm 依存に `git+ssh://` 形式のパッケージがあり、git 未インストールでビルド失敗。`git`, `python3`, `make`, `g++` を追加。

**2. SSH で GitHub に接続できない**
Docker 内に SSH 鍵がないため `git+ssh://` が失敗。`.gitconfig` の `insteadOf` で SSH → HTTPS にリライト。`git config --global` を2回呼ぶと上書きされるため、`printf` で直接書くのがポイント。

**3. SSL 証明書エラー**
`ca-certificates` が `slim` イメージに入っていなかった。apt で追加。

### コンテナ起動編

**4. `openclaw start` が存在しない**
正しくは `openclaw gateway --port 18789`。

**5. ホームディレクトリがない**
`useradd -r` はホームを作らない。`-m` フラグで解決。

**6. Dashboard にアクセスできない**
Gateway が `127.0.0.1` にバインドしており、Docker 外からアクセス不可。`--bind lan` で `0.0.0.0` にバインド。加えて `allowedOrigins` の設定が必要。

**7. pairing required**
OpenClaw はブラウザをデバイスとして認識し、CLI でペアリング承認が必要。コンテナ再ビルドのたびに再承認が必要になる。

### モデル設定編

**8. Anthropic API キーが必要と言われる**
デフォルトモデルが `anthropic/claude-*` だった。`openclaw.json` で `ollama/qwen3.5:9b` に変更。

**9. Ollama プロバイダーが未登録**
環境変数 `OLLAMA_API_KEY` だけでは不十分。`models.providers.ollama` に `models` 配列を含む完全な設定が必要。

**10. `docker compose run --rm` で書いた設定が消える**
一時コンテナのファイルシステムに書き込まれるため、コンテナ終了で消失。Dockerfile にベイクするか、`COPY` で解決。

### モデル選定編

**11. Gemma 3 がツール呼び出し非対応**
`gemma3:12b` は Ollama のネイティブ tool calling に対応していない（12B/27B とも）。OpenClaw はツール呼び出し必須のため、対応モデルへの切り替えが必要だった。

### メモリ・パフォーマンス編

**12. Docker 内 Ollama でメモリ不足**
Docker Desktop の Linux VM 経由だとメモリ効率が悪い。`contextWindow: 131072` で KV cache だけで 5.2 GiB 必要になり OOM。Ollama を Mac ローカルに移行して解決。

**13. コンテキストウィンドウの最低要件**
`contextWindow: 8192` だと OpenClaw の最低要件（16000）を下回る。`32768` → 最終的に `131072` に設定。

**14. 100% context used アラート**
32K トークンではシステムプロンプト＋数往復の会話で枯渇。128GB RAM なら 128K コンテキスト（~12 GiB）で余裕。

### Workspace 編

**15. エージェントが USER.md を認識しない**
`openclaw.json` に `agents.defaults.workspace` が未設定だった。パスを指定して解決。

**16. workspace が read-only で書き込めない**
docker-compose で `:ro` マウントしていたが、OpenClaw は workspace 内に `.openclaw/` を作成する必要がある。read-only を解除し、`.gitignore` に `workspace/.openclaw/` を追加。

## 学び

### Ollama は Docker より Mac ローカルで動かすべき

Docker Desktop on Mac は Linux VM 上で動くため、メモリの割り当てに制限がある。Apple Silicon の統合メモリを活かすには、Ollama はホスト側で動かし、Docker コンテナからは `host.docker.internal` で接続するのがベスト。

### ローカル LLM でのモデル選定は「ツール対応」が最重要

エージェント用途では tool calling（function calling）対応が必須。人気モデルでも対応していない場合がある。

| モデル | ツール対応 | 備考 |
|---|---|---|
| Gemma 3 (12B/27B) | ❌ | コミュニティ版 `orieg/gemma3-tools` なら可能 |
| Qwen 3.5 (9B) | ✅ | ネイティブ対応 |
| Qwen 2.5 (14B) | ✅ | ネイティブ対応 |
| Llama 3.1 (8B) | ✅ | ネイティブ対応 |

### コンテキストウィンドウとメモリのトレードオフ

コンテキストウィンドウを大きくすると KV cache のメモリ使用量が増える。

| contextWindow | KV cache (概算) | 合計メモリ (qwen3.5:9b) |
|---|---|---|
| 8,192 | ~0.3 GiB | ~7 GiB |
| 32,768 | ~1.3 GiB | ~8 GiB |
| 131,072 | ~5.2 GiB | ~12 GiB |

Mac のメモリに余裕があるなら大きく取って問題ない。

### 小さいモデルはシステムプロンプトの遵守が弱い

9B モデルだと「セッション開始時に MEMORY.md を読んで記録せよ」のような複雑な指示を自律的に守らない場面がある。明示的に「メモリに保存して」と指示すれば実行はできる。より大きなモデル（32B+）なら改善が期待できる。

### Dockerfile のコツ

- **`printf` で JSON をベイクするより `COPY` が良い。** 可読性が段違い
- **`git config --global` は上書きに注意。** 同一セクションキーだと後勝ちになる。複数 `insteadOf` を書くなら `printf` で `.gitconfig` を直接書く
- **`useradd -r` はホームディレクトリを作らない。** `-m` を忘れずに
- **シークレットは `.env` + `env_file` で注入。** Dockerfile や docker-compose.yml にベタ書きしない

### workspace の設計思想

```
workspace/
├── 定義ファイル（SOUL.md 等） → エージェントの人格・ルール（人間が書く）
├── MEMORY.md                  → 長期記憶サマリ（エージェントが更新）
└── memory/                    → 日次ログ（エージェントが書き込む）
```

- workspace はホストとマウント共有 → `git push` で GitHub に永続化
- `.openclaw/` はランタイム用 → `.gitignore` で除外
- 定義ファイルは人間が管理、memory はエージェントが管理、という役割分担
