# Code Generation Plan: NOIM変換ツール(詳細設計書に基づく変換の実装)

## Summary

- Builds: 3つの入力CSV(ノード/エンティティ/サービス)を、詳細設計書のとおりNOIM形式のリソース8種に変換し、`resourceInfo_*.csv` として出力するPython CLI
- Touches: `noim_converter.py`(作り直し)、`noim/`(新規パッケージ)、`tests/`(作り直し)
- Tests: 標準ライブラリ unittest。変換ごとの単体テスト、入出力のテスト、実データ(`入力ファイル/`)での通し実行

## 入力資料と優先順位

- 変換ロジックは `NOIM変換ツール_詳細設計書_ハッカソン.xlsx`(以下、設計書)に従う。S3、cron、ロック、カウンタ、ログ通知は対象外。
- 大枠(コマンド形式、エラー時の停止、出力ディレクトリの扱い、期待結果の比較)は `.kiro/specs/noim-converter/requirements.md`(以下、specs)を優先する。
- 出力CSVの形式は `インタフェース仕様書/NOIM変換ツール_ファイル仕様書.xlsx`、各リソースのJSON構造は `インタフェース仕様書/外部インタフェース仕様書.xlsx` に従う。
- 入力CSVの実物は `入力ファイル/` の3ファイル。

## 確認済みの決定(ユーザー回答)

1. 入力は設計書の3種のCSV(`t_mst_vxm_node_info_YYYYMMDDhhmm.csv`、`t_mst_vxm_entity_info_...`、`t_mst_vxm_service_...`)だけ。種類ごとに最新時刻のファイルを使う。入力ディレクトリ直下に3種のどれにも当てはまらないファイルがあればエラー。3種がそろわない場合もエラー。
2. zipは作らない。出力ディレクトリに直接CSVを書く。
3. リソースごとにCSV化して順に書く。書き込みに失敗したら、書き出し済みのファイルは残す。
4. objectIdは `http://localhost:3000/<種別>/system_` + 10進8桁。種別ごとに1から採番し、実行のたびに1から振り直す。LRSpec/PRSpecは固定値(設計書の `spec_00000001`〜`spec_00000008`)。
5. LAG層のXCは、ノードのA側/Z側ごとに1つを共有する。2件目以降のサービスのCPは、そのXCの端点に追加する。
6. ノードのプロファイル名のキーは、設計書および仕様書どおり `nodeProfileName`。
7. PDの `nttLadder` は、CEのとき数値の `1`、それ以外は `null`。
8. LRSpec/PRSpecのCSV(`resourceInfo_lrSpecs.csv`、`resourceInfo_prSpecs.csv`)は作成しない。出力は8ファイル。
9. 設計書に書かれていない部分の解釈(ユーザー承認済み):
   - LD層TPEの `_physicalPortRefList` は、同じエンティティから作ったPPortのobjectIdにする。
   - 仮想PPortの変換で、LAG層TCPの `nttServerTpeRefList` に、作成したLD層TPE(仮想PPort相当)のobjectIdを入れる。
   - NFDの `_edgeTpeRefList` は、XCの各端点のTPEが持つ `nttServerTpeRefList` を集めたものにする。

## 追加で採用する解釈(設計書の文言が曖昧な箇所。完了報告に載せる)

- 設計書 pstruct_converter の「unique_id確認」で `_set_pport_physical_port` と書かれている箇所は転記ミスとみなし、`unique_id == "全国"` の判定として実装する。
- LD層のTPE/TLは `service_profile_name == "物理(BR)"` のサービスだけを対象にする(設計書の概要 ⑤⑥ に合わせる)。クライアントエンティティが見つからない、または2件でない場合は、サービス名を示してエラーにする(設計書は「何もしない」)。
- LAG層は `service_profile_name == "LSP詳細(BR)"` のサービスだけを対象にする。`client_entity_id_list` が空のサービスは読み飛ばす。2件以外はエラー。
- 設計書の「Z側XC管理マップにA側CPのobjectIdを追加」はZ側CPの誤記とみなす。
- 設計書にある空のリスト項目は、空の配列 `[]` として出力する。CSVの空欄は空文字 `""` のまま扱う。
- 出力CSVのJSONは `json.dumps(..., ensure_ascii=False)` で作り、1行1リソース、キーの順序は設計書の記載順にする。
- 各ファイル内の行順は、種別ごとのリストを変換の実行順につなげたもの(設計書の「リストごとにCSVに変換」に合わせる)。TPEは LD層(物理ポート由来) → LAG層TCP → LAG層CP → LD層(仮想PPort相当)、FREは NC → LC → XC、PPortは エンティティ由来 → 仮想PPort。
- BOMなしのUTF-8、改行はCRLF、全フィールドをダブルクォートで囲む(ファイル仕様書)。入力CSVのBOMは読み込み時に取り除く。

## 実装の構成

- `noim_converter.py`: コマンドライン、全体の流れ、期待結果との比較(エントリーポイント)
- `noim/__init__.py`
- `noim/errors.py`: `ToolError`(利用者に表示するエラー)
- `noim/input_csv.py`: 入力ファイルの選択、CSV読み込み、PostgreSQL配列表記(`{a,b}`、`{"x y","z"}`)のパース、行オブジェクト
- `noim/object_id.py`: objectIdの採番
- `noim/converters.py`: PStruct、PD、PPort、PLink、LD層TPE/TL、LAG層FRE/TPE、LD層TPE(仮想PPort相当)、NFD、仮想PPortの変換
- `noim/output_csv.py`: `resourceInfo_*.csv` の書き出し(一時ファイル経由で置き換え)
- 外部ライブラリは使わない(標準ライブラリのみ)。Python 3.8 で動く構文にとどめる。

## 変換の仕様(設計書から整理)

変換の実行順と採番順は、設計書 converters_handler のとおり: PStruct → PD → PPort → PLink → LD層TPE/TL → LAG層FRE/TPE → LD層TPE(仮想PPort相当) → NFD → 仮想PPort。

共通:
- リソースは辞書(キー順は設計書の記載順)。`_resourceSpecification` は設計書の固定URL。`resourceCapacity` は `[]`。
- nttManagementInfo の項目は、入力CSVの列順どおりに「接頭辞 + 列名のUpperCamelCase」でキー名を作る(node/entity/service)。ただしノードの `node_profile_name` は `nodeProfileName`。リスト項目(`scope_id_list`、`scope_name_list`、サービスの `*_list` 6つ)は配列、それ以外は文字列。
- キー名は外部インタフェース仕様書の一覧と一致させる(テストで確認する)。

PStruct:
- ノードごとに `unique_id` を設定: CE以外は、`unification_bldg_code` が空でなければその値、なければ `parent_node_group_name == "BR網"` のとき `name`、それ以外は `name` を「・」で分けた3番目(3つ未満なら `name`)。CEは `"全国"`。
- `unique_id` が未登録のノードだけPStructを作る。`"全国"` はCE用の固定1件(`objectId` は `.../pStructs/system_00000000`、採番しない)。それ以外は採番して作る。`commonName` と `fullAddress` は、`unification_bldg_code` があれば `build_name`、なければ `BR網` のとき `name`、それ以外は「・」の3番目(なければ `name`)。`gridX/gridY` は緯度/経度(空なら `"0"`)。
- `unique_id → PStructのobjectId` の辞書を返す。

PD: ノードごと。`nttPhysicalStructure` はCEなら `.../pStructs/system_00000000`、それ以外は辞書の値。`nttLadder` はCEなら `1`、それ以外は `null`。
PPort: エンティティごと。採番し、`node_id` が `nodeRaVxmId` と一致するPDの `_hardware._physicalConnector` に `{"_physicalPort": objectId}` を追加する(一致するPDがなければ追加しない)。
PLink: `service_profile_name == "物理(BR)"` のサービスごと。`_physicalConnector` は、`client_entity_id_list` の順に、`entityRaVxmId` が一致するPPortの `{"_physicalPort": objectId}`。

LD層TPE/TL(物理(BR)のサービスごと): クライアントエンティティ2件それぞれからTPEを作り(`ltType: "TCP"`、`layerProtocolNameList: ["logicalDevice"]`、`tpeType: "PTP"`)、A側/Z側のTPEのobjectIdを `_aEndTpeRefList` / `_zEndTpeRefList` に持つTLを作る。採番は サービスごとに TPE(A)、TPE(Z)、TL の順。

LAG層FRE/TPE(LSP詳細(BR)のサービスごと。管理マップは NC: (A側ノード, Z側ノード)、A側XC: A側ノード、Z側XC: Z側ノード):
1. A側/Z側のノードIDは、クライアントエンティティの `node_id`。
2. NCがなければ、A側/Z側それぞれ「XCがなければTCPを新規作成(`commonName` はサービス名)、あればXCの端点のTCPを使う」でTCPを決め、NCを作る。
3. `network_service_id_list` の1件目のサービスから作ったTLを探し、その A/Z 側LD層TPEからLAG層のCPを2件作る。A側/Z側のXCが既にあれば、そのXCの端点にCPを追加する。
4. LCを作る(端点はCP2件、`nttServerConnectionRefList` はTLのobjectId)。
5. A側/Z側のXCがなければ、(TCP, CP) を端点としてXCを作り、管理マップに登録する。NCの `_supportingFreRefList` にA側/Z側のXCのobjectIdを設定する。
採番の順序は サービスごとに TCP(必要なら)、NC(必要なら)、CP(A, Z)、LC、XC(A, Z、必要なら)。

LD層TPE(仮想PPort相当): LAG層TCPごと。TCPの `transmissionParameterList` と `ltType` を引き継ぎ、`layerProtocolNameList: ["logicalDevice"]`、`tpeType: "PTP"`。TCPの `nttServerTpeRefList` にそのobjectIdを設定する。
NFD: XCごと。`commonName` はXCの名前、`_edgeTpeRefList` は上記の解釈のとおり。固定項目は設計書の記載のとおり。
仮想PPort: XCごと。XCの端点のうち、CPの側から対応するPD(CPのLD層TPEの `_physicalPortRefList` のPPortを持つPD。同じPDは1回だけ取得)、TCPの側からLD層TPE(仮想PPort相当)を取得。PPort(`commonName`/`vendorPortName` は `"vPort"`、`nttManagementInfo` は `{}`)を採番して作り、PDの `_hardware._physicalConnector` にその `{"_physicalPort": objectId}` を、TPEの `_physicalPortRefList` にそのobjectIdを追加する。

入出力とエラー(specsを優先):
- 実行形式: `python noim_converter.py <入力ディレクトリ> <出力ディレクトリ> [--expected <期待結果ディレクトリ>]`。終了ステータスは 成功 0、読み込み/変換/書き込み/比較の失敗 1、引数エラー 2。エラーは標準エラー出力へ、ファイル名(分かれば項目名や行番号)を含めて表示し、最初のエラーで止まる。
- 入力は 入力ディレクトリ直下のファイルだけ(サブディレクトリは対象外)。
- 読み込みと変換はすべてメモリ上で終え、成功してから出力を書く。書き込みは一時ファイル(`.<名前>.tmp`)に書いてから置き換え、失敗したら一時ファイルを消す。出力ディレクトリがなければ作る。同名ファイルは上書きする。
- `--expected` を付けると、今回書き出した8ファイルと期待結果ディレクトリ直下のファイルを内容(バイト列)で比較する。差異は読める形(UTF-8として読めれば差分、読めなければ最初の差異位置)で表示し、不一致なら終了ステータス1。付けなければ比較しない。

## 実装ステップ(test-after)

- [x] Step 1: プロジェクト構成の作成。`noim_converter.py`、`noim/`(`__init__.py`、`errors.py`)、`tests/`(`__init__.py`)を用意する。既存の `README.md`、`AGENTS.md`、`.gitignore`、`.aidlc-version`、`入力ファイル/`、`インタフェース仕様書/`、設計書は変更しない。
- [x] Step 2: テスト実行環境の準備。追加パッケージは不要で、`python3 -m unittest discover -s tests -t .` を実行コマンドとして `unit-test-instructions.md` に記録する(設定ファイルは不要)。
- [x] Step 3: 入力とobjectIdの実装(FR1、NFR1、NFR2)。`noim/input_csv.py`(ファイル選択、CSV読み込み、配列表記のパース、行オブジェクト)と `noim/object_id.py`。
- [x] Step 4: 入力とobjectIdのテストを書いて実行する(3種の選択と最新の選択、判別できないファイルのエラー、種類不足のエラー、サブディレクトリの除外、存在しない/ディレクトリでない入力、読み込み失敗、必須列不足、配列のパース、採番)。
- [x] Step 5: PStruct、PD、PPort、PLink、LD層TPE/TLの変換の実装(FR2)。`noim/converters.py`。
- [x] Step 6: 上記の変換のテストを書いて実行する(PStructの一意化とCE、PDの `nttLadder` と物理コネクタ、PPortとPDの紐づけ、PLinkの物理コネクタ、LD層TPE/TLの紐づけ、キー名が外部インタフェース仕様書の一覧と一致すること)。
- [x] Step 7: LAG層FRE/TPE、LD層TPE(仮想PPort相当)、NFD、仮想PPortの変換の実装(FR2)。`noim/converters.py`。
- [x] Step 8: 上記の変換のテストを書いて実行する(TCP/XCの共有とCPの追加、NC・LC・XCの端点と `_supportingFreRefList`、`nttServerTpeRefList`、NFDの `_edgeTpeRefList`、仮想PPortとPD/TPEの紐づけ、採番順)。
- [x] Step 9: 出力と全体の流れの実装(FR3、FR4、FR5)。`noim/output_csv.py`(8ファイルの書き出し)と `noim_converter.py`(`run`、期待結果との比較、`main`)。
- [x] Step 10: 出力、全体の流れ、コマンドラインのテストを書いて実行する(書式: ヘッダ・全項目クォート・CRLF・BOMなし・エスケープ、出力ディレクトリの作成と上書き、書き込み失敗、期待結果の一致/不一致/未指定、引数不足、実データ `入力ファイル/` での通し実行と、参照するobjectIdがすべて出力内に存在すること)。
- [x] Step 11: 環境設定。追加の設定ファイルは作らない(NFR1、NFR2)。
- [x] Step 12: ドキュメントとトレーサビリティ。`code-summary.md` に作成ファイル・採用した解釈・実行したテスト・未確認事項を記録し、`traceability.json` を生成する。

## 要件とステップの対応

| 要件 | 実装 | テスト |
|---|---|---|
| FR1.1〜FR1.5(入力の選択と読み込み。FR1.1/FR1.2は決定1により3種のCSVに変更) | Step 3 | Step 4 |
| FR2.1(設計書の変換) | Step 5, Step 7 | Step 6, Step 8 |
| FR2.2(設計書にない規則を足さない) | Step 5, Step 7 | Step 6, Step 8 |
| FR2.3(入出力の対応: 3入力から8出力) | Step 9 | Step 10 |
| FR2.4, FR2.5(変換エラー) | Step 3, Step 5, Step 7 | Step 4, Step 6, Step 8, Step 10 |
| FR3.1〜FR3.6(書き込み) | Step 9 | Step 10 |
| FR4.1〜FR4.3(実行方法) | Step 9 | Step 10 |
| FR5.1(実データでの1回実行) | Step 9 | Step 10(設計書の変換例の期待結果はないため、実データでの終了ステータス0と出力の整合を確認) |
| FR5.2〜FR5.5(期待結果との比較) | Step 9 | Step 10 |
| NFR1, NFR2 | Step 1, Step 11(標準ライブラリのみ) | Step 10(起動確認) |

## Testing Contract

```json
{
  "version": 1,
  "methodology": "test-after",
  "source": "org",
  "ordering": "implement each applicable testable layer, then write and run that layer's tests.",
  "scope": "express",
  "test_strategy": "minimal",
  "project_type": "greenfield",
  "applicable_notes": [
    {
      "layer": "org",
      "text": "We treat tests as a first-class deliverable in every Bolt. The specific\nmethodology (TDD, BDD, ATDD, or classic test-after) is affirmed at\npractices-discovery and recorded in `team.md` under this heading with explicit\n`Methodology` and `Ordering` fields; Code Generation resolves those fields\nindependently from coverage, tooling, and scope notes.\n\nWhen no posture has been affirmed, our default per scope is:\n- **Methodology**: test-after\n- **Ordering**: implement each applicable testable layer, then write and run\n  that layer's tests.\n- `mvp`, `enterprise`, `feature`, `infra`, `classic` add an 80% line-coverage\n  floor and CI execution before merge.\n- `bugfix`, `security-patch` add a targeted regression for the specific\n  bug/vulnerability and require the existing suite to remain green.\n- `express` uses the Minimal strategy: requirement-driven unit tests (one per\n  requirement, with a happy-path floor per component); existing tests remain\n  green.\n- `poc`, `refactor`, `workshop` add no extra new-test floor and require the\n  existing suite to remain green.\n\nThe active `Test Strategy` still applies in every scope and determines test\nvolume/types. Scope floors are additive; they never reduce or replace the\nselected strategy.\n\nBuild and Test verifies defined coverage floors and affirmed quality targets;\nthey may not be weakened to make a step pass.\n\nAffirm a stricter posture in `team.md` if the team commits to one."
    }
  ],
  "obligations": {
    "strategy": "minimal",
    "strategy_volume": [
      "One verifiable test per requirement at the narrowest effective level.",
      "At least one happy-path unit test per component.",
      "Unit tests are the default; a bugfix/security scope floor may require an integration or E2E regression when that is the narrowest level that reproduces the defect."
    ],
    "scope_floor": [
      "Keep the existing test suite green.",
      "This scope adds no extra new-test floor beyond the selected test strategy."
    ],
    "combination_rule": "Apply every selected-strategy obligation and every scope-floor obligation; neither replaces the other, and a targeted scope regression may add the narrowest necessary test type beyond the strategy default."
  },
  "plan_profile": {
    "methodology": "test-after",
    "runner_step": "Bootstrap the minimal test runner/configuration and record the exact unit-scoped command.",
    "runner_ready_before_first_test": true,
    "testable_layers": [
      "Data model / database behavior",
      "Repository / data access",
      "Business logic",
      "API / endpoint",
      "Frontend behavior"
    ],
    "steps": [
      "Project structure and production configuration skeleton.",
      "Bootstrap the minimal test runner/configuration and record the exact unit-scoped command.",
      "Data model / database behavior - implement.",
      "Data model / database behavior - write and run its tests after implementation.",
      "Repository / data access - implement.",
      "Repository / data access - write and run its tests after implementation.",
      "Business logic - implement.",
      "Business logic - write and run its tests after implementation.",
      "API / endpoint - implement.",
      "API / endpoint - write and run its tests after implementation.",
      "Frontend behavior - implement.",
      "Frontend behavior - write and run its tests after implementation.",
      "Environment/build configuration.",
      "Documentation and traceability."
    ]
  },
  "input_sha256": "sha256:6cb23162168334768ff8f29699669845616670119c9eb58bf9bb05af6fefd3de",
  "contract_sha256": "sha256:283a836cf0757ed2efa4b5dd3135f3986eb59880b3017f4ba87a685cfebedfdb"
}
```
