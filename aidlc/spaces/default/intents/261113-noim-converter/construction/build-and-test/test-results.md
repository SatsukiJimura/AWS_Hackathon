# テスト結果: NOIM変換ツール

実行日時: 2026-11-13T04:54:35Z(UTC)  / 環境: Python 3.14.2(Linux)

## ビルド

- コマンド: `python3 -m py_compile noim_converter.py noim/__init__.py noim/errors.py noim/input_csv.py noim/object_id.py noim/converters.py noim/output_csv.py`
- 結果: 成功(終了ステータス 0)

## 単体テスト

- コマンド: `python3 -m unittest discover -s tests -t . -v`(ワークスペース直下で 1 回)
- 結果: **42 件実行、42 件成功、失敗 0、エラー 0、スキップ 0**(終了ステータス 0)

| テストファイル | 件数 | 対象 |
|---|---|---|
| `tests/test_input.py` | 16 | 入力ファイルの選択、CSV読み込み、配列表記のパース、objectId の採番 |
| `tests/test_converters.py` | 15 | PStruct、PD、PPort、PLink、LD層TPE/TL、LAG層FRE/TPE、NFD、仮想PPort |
| `tests/test_output.py` | 11 | CSV書式、書き込み失敗、`--expected`、引数エラー、実データでの通し実行 |

stage-level の `unit-test-instructions.md` が持つコマンドは上の 1 本だけ。Unit 別の命令ファイルはない(zero-Unit)。

## 実データでの通し実行(FR5.1)

- テスト `tests.test_output.RealDataTest.test_end_to_end_via_command_line` が `入力ファイル/` の 3 ファイルを一時ディレクトリへ出力し、成功した。
- 確認内容: 終了ステータス 0、8 ファイルの生成、ヘッダ `"objectId","data"`、全フィールドのクォート、CRLF、BOMなし、各行の `data` が JSON、objectId の重複なし、参照先の objectId がすべて出力内にある。
- 設計書に変換例(期待結果)がないため、期待結果との一致は確認できていない。

## カバレッジ

- カバレッジの下限は設定されていない(express の Minimal)。カバレッジ計測は実行していない。

## 失敗の詳細

なし。

## Target Verification Matrix

`build-and-test-summary.md` の同名の表を参照。

## Loop-Back Log

なし(ループバックは発生していない)。
