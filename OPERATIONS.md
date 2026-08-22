# Season 0 運用手順

## Migration / deploy

1. PostgreSQLを `pg_dump --format=custom` でバックアップし、別DBへ `pg_restore` できることを確認する。
2. `migrations/001_season_0.sql`、`migrations/002_atomic_integrity.sql` の順で `psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f <file>` を実行する。既存の `economy_users`、残高、inventory、ランキングは削除・再作成しない。
3. `.env.example` の新規変数をWeb/Botそれぞれのsecret storeへ設定する。OAuth redirect URIは本番HTTPS URLと完全一致させる。
4. Botを先にdeployし `/health`、Discord `/status`、`/history` を確認する。次にWebをdeployしOAuth、state取得、1回の操作を確認する。
5. 一般公開確認後にだけPatch NotesのWebステータスを「一般公開 / メインクライアント」へ変更する。

## Rollback / forward fix

アプリは旧テーブルを残すforward migrationなので、直前のBot/Webイメージへ戻せる。新テーブルはrollback中も削除しない。新しいSeason操作が既にある場合、データを消すdown migrationは実行せず、修正版をforward deployする。アイテム価格・確率だけを旧版へ戻すとWeb表示と実抽選がずれるため、Bot/Webを同じreleaseへ揃える。

## Season reset

`npm run season:reset` はdry-runで対象人数、Lv30人数、EXPチップ数だけを表示する。結果を保存してから `SEASON_RESET_OPERATOR=<実行者ID> npm run season:reset -- --execute` を1回実行する。Botの全ユーザー更新と同じPostgreSQL advisory lockを取得してから未finalize行をlockするため、実行中のゲーム操作とは直列化される。EXPチップとSeason 0パッシブの明示allowlistだけをinventoryから除去し、再実行時は0件となる。永久称号・season completion・通常残高・通常アイテムは保持される。

## Log retention

`npm run logs:prune` は既定120日より古い `game_action_logs` の対象件数だけを表示する。確認後、`LOG_RETENTION_OPERATOR=<実行者ID> npm run logs:prune -- --execute` で1,000件ずつ削除する。実行結果自体は新しい監査ログへ記録される。`season_completions`、`user_titles`、`level_reward_grants` は保持対象であり削除しない。metadataへtoken、Cookie、OAuth secretを保存しない。

## Backup / restore verification

毎週custom-format backupを取得し、月1回は隔離DBへrestoreする。件数、残高合計、inventoryを持つユーザー数、Season progress件数、Lv30称号件数を本番backup元と照合する。復旧時はWeb/Botを停止し、DBをrestoreしてからBot→Webの順で起動する。
