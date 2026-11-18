## Developer Code Scan Results

### Scan Coverage
- **Analyzed deeply**:
  - `./noim_converter.py`
  - `./noim/` (`__init__.py`, `converters.py`, `errors.py`, `input_csv.py`, `object_id.py`, `output_csv.py`)
  - `./infra/` (`app.py`, `noim_stack.py`, `cdk.json`, `requirements.txt`, `.gitignore`, `README.md`)
  - `./入力ファイル/`(CSV 3 本。ヘッダ・件数・値の分布を読み取り専用で集計)
  - `./README.md`、`./.gitignore`
- **Skimmed only**:
  - `./tests/`(`test_converters.py`、`test_input.py`、`test_output.py`。ファイル名とテスト関数の数(42)だけ確認。中身は読んでいない)
  - `./インタフェース仕様書/`、`./NOIM変換ツール_詳細設計書_ハッカソン.xlsx`(開いていない。出力構造は `converters.py` と `output_csv.py` で十分に分かったため)
  - `./noim-deploy.zip`(配布物なので開いていない)
- **Left out**: `aidlc/`、`.kiro/`、`.git/`、`.aidlc-version`、`AGENTS.md`、`__pycache__/`(ルート・`noim/`・`tests/`・`infra/`)、`.venv/`、`cdk.out/`

### Packages Found
- `noim` — Python パッケージ(ライブラリ) — Python 3(標準ライブラリのみ) — 入力 CSV の読み込み、NOIM リソースへの変換、出力 CSV の書き出し
- `noim_converter.py` — CLI スクリプト(入口) — Python 3 — `run()` が 読み込み → 変換 → 書き出し → (任意)期待結果との比較 を順に行う
- `infra` — CDK アプリ — Python(aws-cdk-lib) — 変換ツールを動かす EC2 環境(スタック 1 つ)
- `tests` — unittest 形式のテスト — Python — 入力・変換・出力のテスト

### Build System
- **Type**: 本体にビルドなし(Python スクリプトを直接実行)。`infra` は AWS CDK(Python)。
- **Config Files**: `infra/cdk.json`(`"app": "python3 app.py"`)、`infra/requirements.txt`(`aws-cdk-lib>=2.140.0,<3.0.0`、`constructs>=10.0.0,<11.0.0`)。本体用の `requirements.txt` / `pyproject.toml` / `Makefile` / `Dockerfile` は無い。
- **Build Dependencies**: `infra/noim_stack.py` はリポジトリ全体を CDK アセットとして S3 に上げる(`ASSET_EXCLUDES` で `infra`、`tests`、`入力ファイル`、`*.md`、`*.xlsx` などを除く)。EC2 起動時に `/opt/noim` へ展開される。

### APIs Discovered
- CLI — `noim_converter.py` — `python noim_converter.py <入力DIR> <出力DIR> [--expected <DIR>]`。終了コード 0 成功 / 1 失敗 / 2 引数エラー
- 内部の関数 API — `noim/converters.py` — `convert_all(nodes, entities, services)` が `{種類: [辞書]}` を返す。他に `convert_pstruct`、`convert_pd`、`convert_pport`、`convert_plink`、`convert_ld_tpe_tl`、`convert_lag`、`convert_ld_tpe_vport`、`convert_nfd`、`convert_vport`
- ファイル契約 — 入力 `t_mst_vxm_{node_info,entity_info,service}_<12桁>.csv` / 出力 `resourceInfo_<種類>.csv` × 8(`objectId`, `data`(JSON))。構造の詳細は `api-documentation.md`
- HTTP / REST API — なし
- シェルコマンド — EC2 上の `/usr/local/bin/noim-run`(UserData が作る。S3 `input/` を取得 → 変換 → S3 `output/` へ戻す)

### Frameworks & Libraries
- Python 標準ライブラリ — `argparse`、`csv`、`io`、`json`、`os`、`re`、`sys`、`difflib` — CLI・CSV・JSON・ファイル操作
- `aws-cdk-lib` — `>=2.140.0,<3.0.0` — `infra` のみ
- `constructs` — `>=10.0.0,<11.0.0` — `infra` のみ
- 本体の Python バージョンは未指定。`os.replace` と `os.makedirs(exist_ok=True)` を使うので 3.3 以上が必要。EC2 は Amazon Linux 2023 の `python3`。

### Test Coverage
- **Test Directories**: `tests/`
- **Test Frameworks**: `unittest`(`tests/test_converters.py` の先頭の import で確認。それ以外の中身は読んでいない)
- **Coverage Config**: absent(`.coveragerc`、`pytest.ini`、`tox.ini` なし)

### Code Quality Indicators
- **Linting**: なし(`.flake8`、`ruff.toml`、`pyproject.toml` なし)
- **CI/CD**: なし(`.github/` なし。デプロイは人が CloudShell から `cdk deploy`)
- **Documentation**: ルート `README.md` は 2 行のみ。`infra/README.md` が詳しい(構築・動作確認・後片付け)。各モジュールに日本語の docstring とコメントがあり、質は高い。設計書は xlsx。

### Technical Debt Signals
- 出力 `data` に認証情報の列が入る: `converters.management_info()` が入力の全列(`username`、`password`、`telemetry_username`、`telemetry_password` を含む)を `nttManagementInfo` に写す(`noim/converters.py` の `management_info`、`convert_pd`)。サンプル入力ではこの 4 列はすべて空。実運用データでは出力に認証情報が入りうるので、可視化側で表示・保存しないこと。
- `objectId` は実行のたびに 1 から採番し直す(`noim/object_id.py`)。実行ごとに同じ装置の `objectId` が変わりうるため、アラームとの突き合わせに `objectId` は使えない。
- `objectId` の形式は `http://localhost:3000/<種類>/system_<8桁>` で、URL として解決できない識別子。
- 種類の判別が暗黙: LD 層 TPE は 2 通りある(物理(BR)由来と、仮想 PPort 相当)。どちらも `ltType` が `"TCP"`、`layerProtocolNameList` が `["logicalDevice"]`。判別は `deviceName` が空かどうか、または `_physicalPortRefList` の先がどの PPort か、で行う(`converters.py` の `convert_ld_tpe_tl` と `convert_ld_tpe_vport`)。
- 出力 CSV の 1 行は JSON 全体を 1 フィールドに入れる(`output_csv.to_csv_bytes`)。ファイルは `"objectId","data"` の 2 列・全フィールド引用・CRLF。
- 開いたファイルをテストしていない(スキャンは読むだけの約束)ため、テストの網羅度は未確認。
- インフラは PoC 設定: S3 の `RemovalPolicy.DESTROY` + `auto_delete_objects=True`、EC2 の `user_data_causes_replacement=True`、アラームの通知先なし(`infra/noim_stack.py`)。
- TODO / FIXME / `noqa` はソースに無し。

## Handoff Summary
- **Intent-relevant finding**: 可視化の入力になるのは `resourceInfo_*.csv` の 8 ファイルのうち、装置は `pds`(PD)、装置間の接続は `pls`(PLink)と `tls`(TL)。`pls` の `_physicalConnector[]._physicalPort` が 2 つの PPort(`pPorts`)を指し、各 PPort は PD の `_hardware._physicalConnector[]._physicalPort` から逆引きして装置に着く。`tls` は `_aEndTpeRefList` / `_zEndTpeRefList` から TPE(`tpes`)を経て、`_encapsulatedLtList[0].transmissionParameterList.deviceName` で装置名が直接取れる。アラームと装置の突き合わせの候補キーは PD の `commonName`(入力の `name`、26 件すべて一意)、`nttManagementInfo.nodeRaVxmId`、`nodeHostName`(14 件のみ値あり)、`nodePrimaryIpv4Address`(14 件のみ値あり)。根拠: `noim/converters.py` の `convert_pd`、`convert_plink`、`convert_ld_tpe_tl`。
- **Risks / follow-up**: (1) 上記の認証情報の列。(2) `objectId` は実行ごとに変わるので、アラームの装置キーに使わない。(3) アラームファイル(S3、JSON)の仕様はこのリポジトリに無い。装置のどの項目と突き合わせるかを後続の設計で決める必要がある。(4) 既存スタックの S3 バケットは `input/` と `output/` のみ許可の設計で、`BLOCK_ALL`・`enforce_ssl`・SSE-S3。可視化用のアプリ・アラーム用の置き場は既存スタックに無い。(5) サンプル入力は装置 26 件・エンティティ 463 件・サービス 51 件(物理(BR) 24、VPN回線(BR) 19、LSP詳細(BR) 8)の小規模データ。座標は全装置 `"0"` で、位置情報は地図表示に使えない。
