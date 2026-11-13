# 要件の対応状況(最終確認): NOIM変換ツール

`aidlc engine sensor-traceability --stage build-and-test` の出力に基づく。

**判定: 合格**(対応していない要件なし)。このワークスペースは zero-Unit のため、対応先はすべて Code Generation のステージ単位。

| 要件 | 内容 | 対応先 | 担当 |
|---|---|---|---|
| FR1.1〜FR1.5 | 入力の選択と読み込み | `noim/input_csv.py` | code-generation |
| FR2.1 | 設計書どおりの変換 | `noim/converters.py` | code-generation |
| FR2.2 | 設計書にない規則を足さない | `noim/converters.py` | code-generation |
| FR2.3 | 入出力の対応(3 入力から 8 出力) | `noim_converter.py` | code-generation |
| FR2.4 | 変換エラー | `noim/converters.py` | code-generation |
| FR2.5 | 変換エラーの表示 | `noim_converter.py` | code-generation |
| FR3.1〜FR3.6 | 出力の書き込み | `noim/output_csv.py` | code-generation |
| FR4.1〜FR4.3 | 実行方法 | `noim_converter.py` | code-generation |
| FR5.1 | 実データでの 1 回実行 | `tests/test_output.py` | code-generation |
| FR5.2〜FR5.5 | 期待結果との比較 | `noim_converter.py` | code-generation |
| NFR1 | 標準ライブラリのみ | `noim_converter.py` | code-generation |
| NFR2 | EC2 上で動く(特殊な環境依存を避ける) | `noim_converter.py` | code-generation |

対応していない要件: なし。

## 補足

- FR5.1 は「設計書の変換例を 1 回の実行で変換する」確認。設計書に変換例(期待結果)がないため、実データでの終了ステータス 0 と出力の整合までを確認した。
- NFR2 のうち、EC2 上の Python バージョンでの実行は未確認(Deployment で確認する)。
