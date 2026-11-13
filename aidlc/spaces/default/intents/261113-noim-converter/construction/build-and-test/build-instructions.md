# ビルド手順: NOIM変換ツール

## 前提

- Python 3(確認に使ったのは 3.14.2。コードは Python 3.8 で動く構文にとどめている)
- 追加パッケージは不要(標準ライブラリのみ)
- 環境変数や設定ファイルは不要

## 依存関係のインストール

なし。`pip install` は不要。

## ビルド

コンパイルや、まとめる処理はない。ソースをそのまま実行する。

構文の確認(ビルドの代わり)を、ワークスペース直下で次のコマンドで行う。

```bash
python3 -m py_compile noim_converter.py noim/__init__.py noim/errors.py noim/input_csv.py noim/object_id.py noim/converters.py noim/output_csv.py
```

終了ステータス 0 なら成功。

## 動作確認

テストを実行する。

```bash
python3 -m unittest discover -s tests -t . -v
```

変換を実行する(入力ディレクトリ直下に 3 種の CSV が必要)。

```bash
python3 noim_converter.py <入力ディレクトリ> <出力ディレクトリ> [--expected <期待結果ディレクトリ>]
```

## EC2 での実行

- EC2 に `noim_converter.py` と `noim/` をそのまま配置し、`python3` で実行する。
- 配置は Operation フェーズ(Deployment)で扱う。

## よくある問題

| 症状 | 原因と対処 |
|---|---|
| `ModuleNotFoundError: No module named 'noim'` | ワークスペース直下(`noim_converter.py` と同じ場所)から実行していない。そのディレクトリで実行する |
| 入力ディレクトリのエラー | 直下に `t_mst_vxm_node_info_*.csv`、`t_mst_vxm_entity_info_*.csv`、`t_mst_vxm_service_*.csv` の 3 種が必要。3 種のどれにも当てはまらないファイルがあるとエラーになる |
| 構文エラー | Python 2 では動かない。`python3` を使う |
