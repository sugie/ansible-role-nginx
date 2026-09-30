# MySQL ユーザー向け PostgreSQL CLI チートシート

長年 MySQL と `mycli` を使ってきた人が、PostgreSQL と `pgcli` / `psql` に移行するための早見表です。

## 最初に覚える違い

- `SELECT ...;` のような **SQL はセミコロンで終える**。
- `\dt` のような **バックスラッシュコマンド（メタコマンド）はセミコロン不要**。
- メタコマンドはサーバーの SQL ではなく、クライアントが解釈する。ここでは `psql` の標準コマンドを基準にしている。`pgcli` でも主要なものは利用できるが、挙動が違う場合は `\?` で確認する。
- PostgreSQL では、1 サーバー内の別データベースを `db.table` のように参照できない。別 DB へ移るときは接続を切り替える。
- テーブル名は通常 `schema.table`。未指定時は `search_path` に従い、多くの環境では `public` スキーマが使われる。

## 接続

```bash
# ローカルまたは環境変数で指定されたサーバー
pgcli mydb
psql mydb

# 接続先を明示
pgcli -h db.example.com -p 5432 -U appuser -d mydb
psql  -h db.example.com -p 5432 -U appuser -d mydb

# URI 形式（パスワードを履歴やプロセス一覧へ残さないこと）
pgcli 'postgresql://appuser@db.example.com:5432/mydb'
```

よく使う接続用環境変数は `PGHOST`、`PGPORT`、`PGUSER`、`PGDATABASE`、`PGPASSWORD` である。常用パスワードは `PGPASSWORD` より、権限を `0600` にした `~/.pgpass` の利用を検討する。

## まず覚えるメタコマンド

| 目的 | PostgreSQL (`psql` / `pgcli`) | MySQL で近い操作 |
|---|---|---|
| ヘルプ（メタコマンド） | `\?` | `help` |
| SQL 構文のヘルプ | `\h`、`\h CREATE TABLE` | `help CREATE TABLE` |
| 接続情報 | `\conninfo` | `status` |
| データベース一覧 | `\l` または `\list` | `SHOW DATABASES;` |
| DB を切り替える | `\c mydb` または `\connect mydb` | `USE mydb;` |
| スキーマ一覧 | `\dn` | 直接の対応なし |
| テーブル一覧 | `\dt` | `SHOW TABLES;` |
| 全スキーマのテーブル | `\dt *.*` | 全 DB の一覧に近い |
| テーブル定義を表示 | `\d foo` | `DESCRIBE foo;` |
| 詳細なテーブル定義 | `\d+ foo` | `SHOW FULL COLUMNS FROM foo;` に近い |
| インデックス一覧 | `\di` | `SHOW INDEX FROM foo;` |
| ビュー一覧 | `\dv` | `SHOW FULL TABLES ...` |
| マテリアライズドビュー一覧 | `\dm` | 直接の対応なし |
| シーケンス一覧 | `\ds` | 直接の対応なし |
| 関数・プロシージャ一覧 | `\df`、`\df+ name` | `SHOW FUNCTION STATUS;` など |
| ロール一覧 | `\du` または `\dg` | `SELECT ... FROM mysql.user;` |
| 権限一覧 | `\dp` または `\z` | `SHOW GRANTS;` に近い |
| 表示を縦型に切り替える | `\x` | `\G` |
| 実行時間を表示 | `\timing on` | `mycli` の timing に相当 |
| 終了 | `\q` | `quit` / `exit` |

`S` を加えるとシステムオブジェクトも表示し、`+` を加えると詳細を表示するコマンドが多い。たとえば `\dtS+` はシステムテーブルを含む詳細一覧になる。

パターン指定もできる。

```text
\dt public.*       -- public スキーマのテーブル
\dt *user*         -- 名前に user を含むテーブル
\d public.orders   -- スキーマを明示して確認
```

## `SHOW CREATE TABLE` の代わり

PostgreSQL の `\d+ foo` は列、型、制約、インデックスなどを読みやすく表示するが、元の `CREATE TABLE` 文そのものは返さない。再作成可能な DDL が必要なら `pg_dump` を使う。

```bash
# 指定テーブルのスキーマだけを出力（データは出力しない）
pg_dump -h db.example.com -U appuser -d mydb \
  --schema-only --table=public.foo

# 所有者変更や権限を除き、移植しやすくする例
pg_dump -d mydb --schema-only --table=public.foo \
  --no-owner --no-privileges
```

ビューの定義だけなら、次の SQL でも確認できる。

```sql
SELECT pg_get_viewdef('public.my_view'::regclass, true);
```

## スキーマと `search_path`

MySQL の「database」は、用途によって PostgreSQL の「database」または「schema」に相当する。アプリケーションごとに完全分離するなら database、同じ接続内で名前空間を分けるなら schema を使う。

```sql
-- 現在のスキーマ探索順
SHOW search_path;

-- 現在のスキーマ
SELECT current_schema();

-- セッション中だけ変更
SET search_path TO app, public;
```

接続中の DB やユーザーは次でも確認できる。

```sql
SELECT current_database(), current_user;
```

## 日常の表示・入出力

```text
\x auto                 -- 横に長い結果だけ縦表示
\pset pager off         -- pager を無効化
\pset null '(null)'     -- NULL を空欄ではなく明示
\o result.txt           -- 以後の結果をファイルへ出力
\o                      -- ファイル出力を終了
\i script.sql           -- SQL ファイルを実行
\e                      -- エディターで現在のクエリーを編集
\watch 2                -- 直前のクエリーを 2 秒ごとに再実行
```

CSV の入出力では、サーバー側の `COPY` とクライアント側の `\copy` を区別する。手元のファイルを扱う普段の操作には `\copy` が便利である。

```text
\copy public.foo TO 'foo.csv' WITH (FORMAT csv, HEADER true)
\copy public.foo FROM 'foo.csv' WITH (FORMAT csv, HEADER true)
```

## トランザクション

PostgreSQL も通常は autocommit で、各 SQL 文が個別のトランザクションになる。まとまった変更は明示的に囲む。

```sql
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;
-- 取り消す場合は ROLLBACK;
```

プロンプトが `mydb=!>` のようになったら、トランザクション内でエラーが発生して中断状態になっている。原因を確認して `ROLLBACK;` する。

## 調査でよく使う SQL

```sql
-- PostgreSQL のバージョン
SELECT version();

-- 現在実行中の接続・クエリー（権限により見える範囲が異なる）
SELECT pid, usename, datname, state, query_start, query
FROM pg_stat_activity
ORDER BY query_start;

-- ユーザーテーブルのおおよその行数とサイズ
SELECT
  schemaname,
  relname,
  n_live_tup,
  pg_size_pretty(pg_total_relation_size(
    format('%I.%I', schemaname, relname)::regclass
  )) AS total_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(
  format('%I.%I', schemaname, relname)::regclass
) DESC;
```

クエリー計画は `EXPLAIN`、実測値を含める場合は `EXPLAIN ANALYZE` を使う。後者はクエリーを**実際に実行する**ため、`UPDATE` や `DELETE` では特に注意する。

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 42;
```

## MySQL から移る際につまずきやすい点

1. **識別子の大文字・小文字**: 引用符なしの名前は小文字に畳まれる。通常は小文字の `snake_case` を使う。`"CamelCase"` のように作ると、以後も二重引用符が必要になる。
2. **文字列の引用**: 文字列は単一引用符 (`'text'`)、識別子は二重引用符 (`"column"`)。バッククォートは使わない。
3. **`LIMIT`**: `LIMIT 10 OFFSET 20` を使う。MySQL の `LIMIT 20, 10` は移植しない。
4. **自動採番**: 新規設計では `GENERATED ... AS IDENTITY` を優先する。既存システムでは `serial` もよく見かける。
5. **真偽値**: `boolean` 型と `TRUE` / `FALSE` を使う。
6. **日時**: タイムゾーンを扱う時刻には通常 `timestamp with time zone` (`timestamptz`) を検討する。
7. **更新・削除の結合**: 構文が異なり、`UPDATE ... FROM ...`、`DELETE ... USING ...` を使う。
8. **UPSERT**: `INSERT ... ON CONFLICT (...) DO UPDATE ...` を使う。
9. **暗黙の型変換**: MySQL より厳密。数値と文字列などを安易に比較せず、型を揃える。
10. **DDL とトランザクション**: 多くの DDL をトランザクション内で実行してロールバックできる。

## 最短の練習手順

以下を上から順に手で打つと、日常操作の流れを覚えやすい。

```text
\conninfo
\l
\c mydb
\dn
\dt
\dt public.*
\d public.foo
\d+ public.foo
\di
\x auto
\timing on
\?
\q
```

まずは **`\l` → `\c` → `\dn` → `\dt` → `\d` → `\x` → `\q`** を体に入れ、DDL が必要なときだけ `pg_dump --schema-only --table=...` を使うのが、MySQL からの移行では分かりやすい。
