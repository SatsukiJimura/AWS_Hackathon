# Code Generation Plan: NOIM構成図の可視化とアラーム装置の見える化

## Summary

- Builds: NOIM変換ツールの出力CSVとアラームJSONを読み、構成図(装置=点、接続=線)とアラーム装置の強調を含む1つのHTMLを作るPythonスクリプト
- Touches: `visualizer.py`(新規)、`noim_viz/`(新規パッケージ)、`tests/test_visualizer.py`(新規)、`アラーム/alarm_spec.md` と `アラーム/alarm_sample.json`(装置名を使う形に更新)
- Tests: 標準ライブラリ unittest。要件ごとの単体テスト約14件と、実データ(`入力ファイル/` を変換した出力)での通し実行1件

## 入力資料

- 要件: `aidlc/spaces/default/intents/261113-noim-visualizer/inception/requirements-analysis/requirements.md`(FR1〜FR6、NFR1〜NFR4)
- 出力CSVの構造と参照関係: `aidlc/spaces/default/codekb/AWS_Hackathon/api-documentation.md`(「各リソースの `data`」と「Reference Map」)
- アラームJSON: `アラーム/alarm_spec.md`、`アラーム/alarm_sample.json`
- 既存の変換ツール(`noim_converter.py`、`noim/`)、既存のテスト(42件)、`infra/` は変更しない。

## 設計の決定

| 項目 | 決定 | 理由 |
|---|---|---|
| 読む出力CSV | `resourceInfo_pds.csv`(装置)、`resourceInfo_pPorts.csv`(PLink の端のポート→装置)、`resourceInfo_pls.csv`(物理接続)、`resourceInfo_fres.csv`(LSPの論理接続) | TL は PLink と1対1で同じ接続(二重に描かない)。FR2.2 の「物理接続」は PLink を元に作る |
| 装置の取り出し | PD 1件 → 1つの点。`id`=`nttManagementInfo.nodeRaVxmId`、`name`=`commonName`、`type`=`nttManagementInfo.nodeNodeType` | api-documentation.md |
| 物理接続 | PLink の `_physicalConnector[]._physicalPort`(PPortのobjectId) → PPort の `nttManagementInfo.entityNodeId` → 装置の `nodeRaVxmId`。2つの異なる装置に着いたものを実線の辺にする | Reference Map |
| LSP接続 | FRE の `nttFreType == "NC"` の `_supportingFreRefList`(XCのobjectId) → そのXCの `transmissionParameterList.raVxmId`(= 装置の `nodeRaVxmId`)。2つの異なる装置に着いたものを点線の辺にする | Reference Map |
| 辺の統合 | 同じ種類(物理/LSP)で同じ2装置を結ぶ接続は、1本の辺にまとめ、本数(`count`)と接続名の一覧を持つ(Open Question の解決) | 見やすさ。サンプルは26装置・約32接続 |
| 描けない接続 | 端が装置に着かない、または2つの異なる装置にならない接続は描かず、警告に件数と理由(種類、接続名)を出す | FR2.4 |
| アラームの結びつけ | `targetDeviceId` と装置の `name` の完全一致 | Q1: A、FR3.1 |
| 装置名の重複 | 装置名が重複していたらエラー(結びつけが曖昧になるため) | 前提 A2 |
| 配置 | Python側で、力指向レイアウトを固定の初期配置と固定回数で計算し、座標をHTMLに埋め込む。同じ入力から同じHTML | FR6.2。JS側は表示・ドラッグ・拡大縮小だけ |
| 認証情報の保護 | 許可リスト方式。HTMLに入れるのは、装置の `id`、`name`、`type`、辺の端点・種類・本数・接続名、アラームの6項目だけ。`nttManagementInfo` の他の項目は読むだけで、HTMLに入れない | FR5.1 |
| HTMLへの埋め込み | データはJSONとして `<script type="application/json">` に埋め込み、`</` と `<!--` をエスケープする。外部の読み込みはしない | FR4.1 |
| 色 | 重大度 1 黄 `#f2c200`、2 橙 `#f28c00`、3 赤 `#d62728`。アラームなしの装置は灰青 | FR4.3 |
| 終了ステータス | 成功 0、入力・処理の失敗 1、引数エラー 2。エラーは標準エラー出力へ `エラー: ...` | FR6.3、変換ツールと同じ |
| 実行形式 | `python3 visualizer.py <NOIM出力ディレクトリ> <アラームJSON> <出力HTML>` | FR6.1 |

## 実装の構成

- `visualizer.py`: コマンドライン(`main`)と全体の流れ(`run`)
- `noim_viz/__init__.py`
- `noim_viz/errors.py`: 利用者に表示するエラー(`VizError`)
- `noim_viz/loader.py`: 出力CSVの読み込み(`objectId`, `data`)、アラームJSONの読み込みと検証
- `noim_viz/graph.py`: 装置・辺の組み立て、アラームの結びつけ、警告、サマリ
- `noim_viz/layout.py`: 力指向レイアウト(標準ライブラリのみ、決定的)
- `noim_viz/html_out.py`: HTML(CSS、JS、埋め込みデータ)の生成
- 外部ライブラリは使わない。Python 3.8 で動く構文にとどめる。

## 実装ステップ(test-after)

- [ ] Step 1: プロジェクト構成の作成。`visualizer.py`、`noim_viz/`(`__init__.py`、`errors.py`)、`tests/test_visualizer.py` の空の枠。既存ファイル(`noim_converter.py`、`noim/`、`infra/`、`tests/test_*.py`、`入力ファイル/` など)は変更しない。
- [ ] Step 2: テスト実行環境の確認。追加パッケージは不要。既存の `python3 -m unittest discover -s tests -t .` が動くことを確認し、この変更のテストの実行コマンド `python3 -m unittest tests.test_visualizer -v` を `unit-test-instructions.md` に記録する。
- [ ] Step 3: 読み込みの実装(FR1)。`noim_viz/loader.py`(出力CSV4種、アラームJSON、検証、エラーにファイル名と理由を含める)。
- [ ] Step 4: 読み込みのテストを書いて実行する(正常系、出力CSVが無い、アラームJSONが壊れている、必須項目が無い、`alarmLevel` 範囲外、`status` 不正)。
- [ ] Step 5: グラフの組み立てとアラームの結びつけの実装(FR2、FR3.1、FR3.2、FR5.1)。`noim_viz/graph.py`。
- [ ] Step 6: 上記のテストを書いて実行する(装置、物理接続、LSP接続、辺の統合、描けない接続の警告、アラームの結びつけと「対応する装置のないアラーム」、装置名の重複エラー、`password` が結果に含まれないこと)。
- [ ] Step 7: レイアウトとHTML生成の実装(FR4、FR5.1、NFR1)。`noim_viz/layout.py`、`noim_viz/html_out.py`(自己完結のHTML、SVGの表示、拡大縮小・ドラッグ、装置クリックで詳細パネル、上部のサマリと警告、重大度の色分け、`recovered` は一覧にだけ薄く表示)。
- [ ] Step 8: 上記のテストを書いて実行する(外部URLを読み込まない、色の割り当て=最も重い `active` の重大度、`recovered` のみの装置は強調されない、サマリの数、同じ入力から同じHTML、埋め込みデータのエスケープ、`password` の値がHTMLに無い)。
- [ ] Step 9: コマンドラインと全体の流れの実装(FR6)。`visualizer.py`。あわせて、`アラーム/alarm_spec.md` と `アラーム/alarm_sample.json` を、`targetDeviceId` に実データの装置名を入れる形に更新する(FR3.3)。
- [ ] Step 10: コマンドラインと実データの通し実行のテストを書いて実行する(引数エラーは終了ステータス2、入力エラーは1でHTMLが作られない、成功時は0。実データ: `入力ファイル/` を既存の変換ツールで一時ディレクトリに変換し、更新後の `アラーム/alarm_sample.json` で可視化。装置26、描けない接続の件数が警告に出る、アラームが該当装置に付く)。
- [ ] Step 11: 環境設定。追加の設定ファイルは作らない(NFR1、NFR4)。
- [ ] Step 12: ドキュメントとトレーサビリティ。`code-summary.md` に作成ファイル・使い方・採用した解釈・実行したテスト・未確認事項を記録し、`traceability.json` を生成する。
- [x] Step 13: 修正(依頼「点同士の距離を調整」「上部のタブがクリックできない」「線がつながっていない点がある」)。(a) 装置のラベルの文字を小さくし、点同士の距離(レイアウトの理想の距離・反発)を広げて、ラベルが重ならないようにする。(b) 上部のサマリのチップ(装置、物理接続、LSP接続、アラーム装置、警告)をクリックできるようにする(装置=全体を表示に戻す、物理接続・LSP接続=その種類の線の表示と非表示の切り替え、アラーム装置=アラームのある装置に注目して他をうす暗くする、警告=警告の一覧を開く)。(c) 線につながっていない装置が、なぜあるのかを実データで調べ(接続がデータに無いのか、読み取りの漏れか)、結果を報告する。読み取りの漏れなら直す。データに接続が無いなら、つながっていない装置を図の脇にまとめて置き、サマリに「接続なし N」を出す(新しい接続の規則は足さない)。
- [x] Step 14: 上記のテストを書いて実行する(ラベルが重ならない最小の距離、チップのクリックで状態が切り替わる、つながっていない装置の数と置き場所、既存のテストが成功のまま)。
- [x] Step 15: `code-summary.md` と `traceability.json` を、修正の内容に合わせて更新する。

## 要件とステップの対応

| 要件 | 実装 | テスト |
|---|---|---|
| FR1.1〜FR1.3 | Step 3 | Step 4 |
| FR2.1〜FR2.5 | Step 5 | Step 6 |
| FR3.1、FR3.2 | Step 5 | Step 6 |
| FR3.3 | Step 9 | Step 10 |
| FR4.1〜FR4.5 | Step 7 | Step 8 |
| FR5.1 | Step 5、Step 7 | Step 6、Step 8 |
| FR6.1〜FR6.3 | Step 9 | Step 10 |
| NFR1 | Step 1、Step 7、Step 11 | Step 8 |
| NFR2 | Step 7 | Step 10(HTMLの生成と、表示の目視確認は人が行う) |
| NFR3、NFR4 | Step 5、Step 7、Step 11(AWSのリソースを作らない) | Step 6、Step 8 |

## Testing Contract

```json
{
  "version": 1,
  "methodology": "test-after",
  "source": "org",
  "ordering": "implement each applicable testable layer, then write and run that layer's tests.",
  "scope": "express",
  "test_strategy": "minimal",
  "project_type": "brownfield",
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
    "runner_step": "Verify the existing test runner/configuration and record the exact unit-scoped command.",
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
      "Verify the existing test runner/configuration and record the exact unit-scoped command.",
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
  "input_sha256": "sha256:c8f84e35190e007d9863fad236154d0269ab9bf633fc2a3ff106ed965ba4a9b9",
  "contract_sha256": "sha256:bb76c1b22f8307be6e5780d50cabab891684449d5b164bd0e825e9e4e5c61e5b"
}
```
