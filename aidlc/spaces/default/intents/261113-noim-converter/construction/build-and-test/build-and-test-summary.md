# Build and Test Summary: NOIM変換ツール

## 全体

- ビルド: 成功(構文チェック、終了ステータス 0)
- 単体テスト: 42 件実行、42 件成功
- 実データでの通し実行: 成功(テスト内)
- 詳細: `test-results.md`、`build-instructions.md`、`cross-unit-traceability.md`

## テストの種類

| 種類 | 作成 | 理由 |
|---|---|---|
| 単体テスト | あり(Code Generation で作成済み) | Minimal の方針。要件ごとに 1 件以上 |
| 結合テスト | なし | Minimal では追加の命令ファイルを作らない |
| 性能・セキュリティ | なし | Minimal。NFR の性能・セキュリティ要件もない |

## カバレッジの見込み

- 行カバレッジの下限は設定されていない(express の Minimal)。計測は実行していない。

## Target Verification Matrix

| Target ID | Source | Expected | Actual | Evidence | Owning Stage | Verdict |
|---|---|---|---|---|---|---|
| TC-REQ-TEST | code-generation-plan.md `## Testing Contract` obligations.strategy_volume(要件ごとに 1 件以上のテスト) | 各 FR/NFR に 1 件以上の検証テストがある | FR1〜FR5 の各グループにテストがある(入力 16、変換 15、出力・通し実行 11) | `test-results.md`、`cross-unit-traceability.md` | build-and-test | Met |
| TC-HAPPY-PATH | 同上(コンポーネントごとに正常系の単体テスト 1 件以上) | 入力、変換、出力、CLI の各コンポーネントに正常系テストがある | 4 コンポーネントすべてに正常系テストあり | `tests/test_input.py`、`tests/test_converters.py`、`tests/test_output.py` | build-and-test | Met |
| TC-SUITE-GREEN | 同上 obligations.scope_floor(既存テストが成功のまま) | 全テスト成功 | 42/42 成功、失敗 0 | `python3 -m unittest discover -s tests -t . -v` の終了ステータス 0 | build-and-test | Met |
| TC-CI-RUN | 同上(選んだテストを CI でマージ前に実行する) | N/A | N/A | 計画に CI Pipeline がない(ci-pipeline は SKIP) | build-and-test | N/A |

`TC-CI-RUN` が `N/A` の理由: 計画は CI Pipeline を実行しないため、この目標は適用外。

## Readiness

- build-ready: はい
- test-ready: はい
- deployment-ready: はい(Deployment 段階へ進める。EC2 上の Python バージョンは未確認)

## 既知の制約・残り

- 設計書に変換例(期待結果)がないため、期待結果との 1 文字単位の一致は未確認。
- ファイル仕様書の出力例は JSON のコロンの後に空白がない(`"lrStatus":1,`)が、出力は既定の区切り(`": "`)。期待結果が手に入ったら最初に確認する。
- EC2 の Python バージョンでの実行は未確認。
