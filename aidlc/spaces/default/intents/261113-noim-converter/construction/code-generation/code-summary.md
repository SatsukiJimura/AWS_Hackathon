# Code Summary: NOIM変換ツール

## 作成したファイル

| ファイル | 内容 |
|---|---|
| `noim_converter.py` | CLI の入口。`run`(全体の流れ)、`--expected` との比較、`main`(終了ステータス 0 / 1 / 2) |
| `noim/__init__.py` | パッケージの印 |
| `noim/errors.py` | 利用者に表示するエラー `ToolError` |
| `noim/input_csv.py` | 入力3種(ノード/エンティティ/サービス)のファイル選択、CSV読み込み、配列表記のパース |
| `noim/object_id.py` | objectId の採番(種別ごとに1から) |
| `noim/converters.py` | PStruct、PD、PPort、PLink、LD層TPE/TL、LAG層FRE/TPE、LD層TPE(仮想PPort相当)、NFD、仮想PPort の変換と、全体を呼ぶ `convert_all` |
| `noim/output_csv.py` | 8ファイルの `resourceInfo_*.csv` の書き出し(BOMなしUTF-8、CRLF、全項目クォート、一時ファイル経由) |
| `tests/test_input.py` | 入力・採番のテスト(16件) |
| `tests/test_converters.py` | 変換のテスト(15件) |
| `tests/test_output.py` | 出力形式、書き込み失敗、`--expected`、引数エラー、実データでの通し実行(11件) |

既存の `README.md`、`AGENTS.md`、`.gitignore`、`.aidlc-version`、`入力ファイル/`、`インタフェース仕様書/`、設計書は変更していない。

## 使い方

```bash
python3 noim_converter.py <入力ディレクトリ> <出力ディレクトリ> [--expected <期待結果ディレクトリ>]
```

- 入力ディレクトリ直下に `t_mst_vxm_node_info_*.csv`、`t_mst_vxm_entity_info_*.csv`、`t_mst_vxm_service_*.csv` が必要。種類ごとに最新時刻のファイルを使う。
- 出力は `resourceInfo_*.csv` の8ファイル(LRSpec/PRSpec は作らない)。
- 終了ステータス: 成功 0 / 読み込み・変換・書き込み・比較の失敗 1 / 引数エラー 2。エラーは標準エラー出力へ出る。
- 標準ライブラリのみ。確認に使った Python は 3.14.2(Python 3.8 で動く構文にとどめた)。
- テスト: `python3 -m unittest discover -s tests -t . -v`

## 実行したテスト

- コマンド: `python3 -m unittest discover -s tests -t . -v`(ワークスペース直下)
- 結果: 42件すべて成功。
- 実データ(`入力ファイル/`)の通し実行: 終了ステータス 0。同じ出力先を `--expected` に指定すると一致して 0。
- 実データの出力件数: NC 8 / LC 8 / XC 11 / NFD 11 / TPE 86 / TL 24 / PPort 474 / PD 26 / PLink 24 / PStruct 6。objectId の重複がないこと、参照先の objectId がすべて出力内にあることを通し実行のテストで確認した。

## 採用した解釈(設計書の文言が曖昧な箇所。プランの「追加で採用する解釈」と同じ)

- pstruct_converter の `_set_pport_physical_port` は転記ミスとみなし、`unique_id == "全国"` の判定にした。
- LD層のTPE/TLは `物理(BR)` のサービスだけが対象。クライアントエンティティが2件でない場合はサービス名つきでエラー(設計書は「何もしない」)。
- LAG層は `LSP詳細(BR)` のサービスだけが対象。`client_entity_id_list` が空なら読み飛ばし、2件以外はエラー。
- 「Z側XC管理マップにA側CPを追加」はZ側CPの誤記とみなした。
- 空のリスト項目は `[]`、CSVの空欄は `""` のまま。
- 行順は変換の実行順にそろえた(TPE は LD層(物理由来) → TCP → CP → LD層(仮想PPort相当))。このため TCP と CP の objectId は番号順に並ばない。採番は変換の実行順どおり。

## 実装時に決めたこと(プランに明記がなかった部分)

- `--expected` の比較: 書き出した8ファイルのそれぞれを、期待結果ディレクトリ直下の同名ファイルとバイト列で比べる。期待側にだけあるファイルは無視する。
- 仮想PPortのPD探索: XCの端点のCPのうち、最初にPDが特定できたものを使う。PDまたは仮想PPort相当のTPEが特定できないXCは、XC名を示してエラーにする。
- NFDの `_edgeTpeRefList`: XCの端点TPEの `nttServerTpeRefList` をそのまま集める。同じLD層TPEを複数のLSPが共有する場合は重複して入る(プランに重複排除の規定がないため除外していない)。
- JSONは `json.dumps(..., ensure_ascii=False)` の既定の区切り(`", "` と `": "`)で作る。

## 実行していないテスト・未確認

- 設計書に変換例(サンプル入力と期待結果)がないため、期待結果との1文字単位の一致は確認できていない。ファイル仕様書の出力例はコロンの後に空白がない(`"lrStatus":1,`)が、例示とみなして既定の区切りにした。期待結果が手に入ったら、この区切りの違いを最初に確認する。
- EC2 の Python バージョンでの実行は未確認(Python 3.8 で動く構文にとどめている)。

## プランからの差異

- なし。Step 1〜11 を実装し、Step 12(このファイルと `traceability.json`)を作成した。ワークスペースは zero-Unit のため `source-manifest.json` は作らない。
