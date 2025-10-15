---
title: "Cloud SQL for PostgreSQL 18 + IAM認証のセットアップ完全ガイド"
emoji: "🐘"
type: "tech"
topics: ["googlecloud", "cloudsql", "postgresql", "iam", "gcp"]
published: false # 公開設定（falseにすると下書き）
---

## はじめに

この記事では、Cloud SQL for PostgreSQL 18のセットアップから、Google Workspace GroupsによるIAM認証の設定までを実践した記録です。本番環境を想定したセキュアな構成を目指しました。

## タスク0: Private Service Accessの設定
### VPCネットワークの確認

```bash
# デフォルトVPC確認
gcloud compute networks list

# なければ作成
gcloud compute networks create default --subnet-mode=auto
```

### Private Service Accessの設定

```bash
# IPアドレス範囲を予約
gcloud compute addresses create google-managed-services-default \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=16 \
  --network=default

# VPCピアリング接続を作成
gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=google-managed-services-default \
  --network=default
```

### 確認

```bash
# VPCピアリング確認
gcloud services vpc-peerings list \
  --network=default \
  --service=servicenetworking.googleapis.com
```

## タスク1: Cloud SQL API有効化

まずはCloud SQL Admin APIを有効化します。

```bash
# Cloud SQL Admin API有効化
gcloud services enable sqladmin.googleapis.com

# 確認
gcloud services list --enabled | grep sqladmin
```

## タスク2: PostgreSQL 18インスタンス作成

```bash
gcloud sql instances create [db-instance-name] \
  --database-version=POSTGRES_18 \      # バージョン指定
  --tier=db-f1-micro \                  # ティアの指定（db-f1-microは最も安いやつ）
  --edition=ENTERPRISE \                # エディションの指定 ENTERPRISE or ENTERPRISE_PLUS
  --region=asia-northeast1 \            # リージョンの指定（asia-northeast1はTokyoリージョン）
  --network=default \                   # ネットワークの指定
  --no-assign-ip \                      # パブリックIPを割り当てない
  --root-password=[YOUR_PASSWORD]       # DBのadminユーザのパスワードを指定
```

**作成結果:**

```
NAME                DATABASE_VERSION  LOCATION           TIER         PRIVATE_ADDRESS  STATUS
[db-instance-name]  POSTGRES_18       asia-northeast1-b  db-f1-micro  10.x.x.x         RUNNABLE
```

### TIPS

editionを指定しないと作成できなかった


## タスク3: 初期データベース作成

```bash
# データベース作成
gcloud sql databases create myapp \
  --instance=[db-instance-name]

# 確認
gcloud sql databases list --instance=[db-instance-name]
```

**結果:**

```
NAME      CHARSET  COLLATION
postgres  UTF8     en_US.UTF8
myapp     UTF8     en_US.UTF8
```

## タスク4: PostgreSQL 18の動作確認

Cloud SQL Studioで接続し、PostgreSQL 18の新機能を確認しました。

```sql
-- バージョン確認
SELECT version();
-- PostgreSQL 18.0 on x86_64-pc-linux-gnu

-- uuidv7()の動作確認
SELECT gen_random_uuid() AS uuid_v4, uuidv7() AS uuid_v7;
```

:::message
**PostgreSQL 18の新機能 `uuidv7()`**
タイムスタンプ順序付きUUIDを生成できます。インデックス効率が向上し、B-treeインデックスでのパフォーマンスが改善されます。
:::

## タスク5: IAM認証設定

### Google Workspace Groupsでセキュリティグループ作成

本番環境を想定し、個人アカウントではなくGoogle Workspace Groupsを使用します。

**作成手順:**

1. **Google Workspace Admin Console**（https://admin.google.com）にアクセス
2. 「ディレクトリ」→「グループ」→「グループを作成」
3. 以下の設定で作成

```yaml
Group name: Developers
Group email: developers@example.com
Description: 開発チーム - Cloud SQLアクセス権限
Access type: Restricted（重要）
Who can join: Only invited users（重要）
Group labels: Security（チェック必須）
```

:::message alert
**セキュリティのポイント**
- `Access type`は`Restricted`に設定
- `Who can join`は`Only invited users`に設定
- 招待されたユーザーのみがCloud SQLにアクセス可能になります
:::

### IAM認証の有効化

```bash
gcloud sql instances patch [myapp-db] \
  --database-flags=cloudsql.iam_authentication=on
```

:::message
この操作はインスタンスの再起動が必要です（1-3分）
:::

### グループをCloud SQLユーザーとして追加

```bash
gcloud sql users create developers@example.com \
  --instance=[myapp-db] \
  --type=CLOUD_IAM_GROUP
```

### IAM認証でログイン

Cloud SQL Studioで以下の手順でログイン：

1. Cloud SQL Studioを開く
2. 「新しい接続」
3. **認証方法**: 「IAM認証」を選択
4. **データベース**: `[myapp-db]`
5. 「接続」

**接続確認:**

```sql
SELECT current_user;
-- user@example.com
```

グループメンバー（`user@example.com`）として接続できました！

### 動作確認

```sql
-- テスト用テーブル作成
CREATE TABLE test_table (
    id SERIAL PRIMARY KEY,
    name TEXT
);

-- 確認
SELECT tablename FROM pg_tables WHERE schemaname = 'public';
-- test_table

-- クリーンアップ
DROP TABLE test_table;
```

## コスト管理

### 開発時の節約方法

使わない時はインスタンスを停止して課金を抑えましょう。

```bash
# インスタンス停止
gcloud sql instances patch myapp-db \
  --activation-policy=NEVER

# インスタンス再開
gcloud sql instances patch myapp-db \
  --activation-policy=ALWAYS
```

## まとめ

Cloud SQL for PostgreSQL 18のセットアップと、本番環境を想定したIAM認証の設定を完了しました。

**達成したこと:**
- ✅ PostgreSQL 18インスタンス作成（ENTERPRISE edition）
- ✅ uuidv7()など新機能の動作確認
- ✅ Google Workspace Groupsでセキュリティグループ作成
- ✅ IAM認証によるCloud SQL Studioへのログイン
- ✅ データベース操作の動作確認

次回はローカル開発環境からの接続と、本格的なスキーマ設計に進みます。

## 参考資料

- [Cloud SQL for PostgreSQL Documentation](https://cloud.google.com/sql/docs/postgres)
- [Cloud SQL IAM Database Authentication](https://cloud.google.com/sql/docs/postgres/authentication)
- [PostgreSQL 18 Release Notes](https://www.postgresql.org/docs/18/release-18.html)
- [Google Workspace Groups](https://support.google.com/a/answer/9400082)
