# Code Quality Assessment

スキャンは読むだけで、テスト、リンタ、プログラムは実行していない。

## Test Coverage

- **Overall**: 計測なし(カバレッジ設定が無い)。
- **Unit Tests**: `tests/test_converters.py`(346 行)、`test_input.py`(181 行)、`test_output.py`(256 行)。`def test_` は合計 42。変換、入力、出力のすべてにテストファイルがある。`test_converters.py` の先頭に、外部インタフェース仕様書の `nttManagementInfo` のキー一覧(`PD_KEYS` など)を持つ。
- **Integration Tests**: ファイルとしては無い。`noim_converter.py` の `--expected` で、実行結果を期待結果ディレクトリとバイト比較できる(期待結果のファイル自体はこのリポジトリに無い)。
- **未確認**: テストの個々の中身と、現在通るかどうか。

## Code Quality Indicators

| 観点 | 評価 | 根拠 |
|---|---|---|
| 命名の明確さ | good | `convert_<リソース>`、日本語の docstring とエラーメッセージ |
| 関数の大きさ | fair | `convert_lag` はクロージャを含め 100 行を超える。他は短い |
| エラー処理 | good | 入力の不整合ごとに `ToolError`(ファイル名・行番号つき)。書き込み失敗時は一時ファイルを消す |
| テストの有無 | good | 3 モジュールすべてにテストファイルあり |
| 重複 | fair | LAG 層の TPE/FRE の組み立てに似た辞書の構築がある(`_lag_tpe`、`_lag_fre` で一部共通化済み) |
| デッドコード | good | TODO / FIXME / `noqa` は無い。未使用の定数は確認していない |
| ファイルの大きさ | fair | `noim/converters.py` は 559 行(500 行超。ただし役割は変換 1 つ) |

- **Linting**: なし。
- **Code Style**: 一貫している(PEP 8 風、docstring は日本語)。
- **Documentation**: ルート `README.md` は 2 行。`infra/README.md` は詳しい。設計書と仕様書は xlsx(今回は未読)。コード中の日本語コメントは、設計書の節との対応を示している。

## Technical Debt

- **認証情報の列を出力に写す**: `converters.management_info()` が入力の全列を写す。`username`、`password`、`telemetry_username`、`telemetry_password` を含む。サンプルは空だが、実データでは出力 CSV と S3 に認証情報が載りうる。可視化側での扱いは要設計(`api-documentation.md` の末尾にも記載)。
- **`objectId` が実行ごとに変わる**: 同じ装置でも出力のたびに別の ID になる(`noim/object_id.py`)。アラームの突き合わせには使えない。
- **暗黙の種類判別**: TPE の 4 種類(`api-documentation.md`)は専用のキーで区別されず、`tpeType`、`ltType`、`layerProtocolNameList`、`deviceName` が空かどうか、の組み合わせでしか区別できない。NFD と XC の対応も作成順か名前頼み。
- **XC の重複**: A 側・Z 側で別々の対応表を使うので、同じノードが両側に出ると XC が 2 つできる(`convert_lag` の `a_xc_map`、`z_xc_map`)。設計書どおりかどうかは未確認。
- **`VPN回線(BR)` を変換しない**: サンプルで 19 件のサービスが、PLink、TL、FRE のいずれにもならない。仕様どおりかは設計書で要確認。
- **PoC 向けのインフラ設定**: S3 の `RemovalPolicy.DESTROY` と `auto_delete_objects=True`、通知先のないアラーム、`user_data_causes_replacement=True`(`infra/noim_stack.py`)。意図的な設定で、README に説明がある。
- **CI/CD なし**: パイプラインは無い。デプロイは手作業。

## Patterns and Anti-patterns

- **良い点**: 入力の検証が細かい(列の有無、列数、配列の形式、2 件でない `client_entity_id_list`)。出力の書き込みは一時ファイル経由。標準ライブラリのみで EC2 に依存を入れずに動く。
- **注意点**: 変換は関数どうしが前段の結果を書き換える(副作用で PD や TPE に追記する)ため、段の順序を変えると結果が変わる。
