# owner_handlers.go: GET /api/owner/chairs の total_distance 計算を効率化

## 問題

`ownerGetChairs` (`webapp/go/owner_handlers.go`) の SQL が `chair_locations` 全件に対して
`LAG` ウィンドウ関数で距離を計算したあとに `owner_id` で絞り込んでいたため、リクエストの
オーナーに無関係な椅子の位置履歴まで毎回計算していた。

## 対応

距離計算用のサブクエリに `WHERE chair_id IN (SELECT id FROM chairs WHERE owner_id = ?)` を追加し、
ウィンドウ関数の計算対象を対象オーナー所有の椅子のみに絞り込んだ。

```sql
FROM chair_locations
WHERE chair_id IN (SELECT id FROM chairs WHERE owner_id = ?)
```

プレースホルダが `chair_id IN (...)` 用と外側の `WHERE owner_id = ?` 用で2つになるため、
Go側のバインド引数も `owner.ID` を2つ渡すように修正 (`owner.ID, owner.ID`)。

## ハマった点

- SQL修正後、Goバイナリを再ビルドせずに `docker compose up` していたため、コンテナは新しく
  起動していても中身のイメージは2026-04-12ビルドの古いものが使われ続け、slow.log にも
  修正前のクエリしか出てこなかった。`docker compose build webapp` でイメージを明示的に
  リビルドする必要がある。
- バインド引数の数を `?` の数と合わせ忘れて `sql: expected 2 arguments, got 1` で
  `GET /api/owner/chairs` が500になり、ベンチマークの初期データチェックで弾かれた。

## 今後の改善案 (未実施)

- `chair_locations` に `(chair_id, created_at)` の複合インデックスがなく、サブクエリ内の
  絞り込み・ソートがフルスキャンになる。追加すると更に効く。
- 位置情報INSERT時に距離を都度加算更新する方式に変えれば、このエンドポイントは単純な
  `SELECT` のみで済み、リクエスト毎のウィンドウ関数計算自体をなくせる。
