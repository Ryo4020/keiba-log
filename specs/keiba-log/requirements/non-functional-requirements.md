# 非機能要件: keiba-log

| 項目 | 内容 |
|---|---|
| 機能名 | `keiba-log` |
| 担当 | nfr-analyst |

## Security

- `REQ-NFR-001` ユーザー間でデータを分離する
- `REQ-NFR-002` 認証が必要な操作を保護する

## Data Integrity

- `REQ-NFR-003` 記録とその関連データ（レース記録・買い目・買い目の集合など親子関係を持つデータ）が不整合な状態にならない

## Availability / Recovery

- `REQ-NFR-004` データ復旧手段を提供する

## Compatibility

- `REQ-NFR-005` スマートフォンとPCで利用できる
