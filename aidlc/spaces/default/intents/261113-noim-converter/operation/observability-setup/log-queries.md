# Log Queries: NOIM変換ツール(dev)

ログは **EC2 のファイル** にあります。CloudWatch Logs には転送していないので、Logs Insights のクエリは使えません。EC2 にセッションマネージャーで接続し、次のコマンドで見ます。

## ログの置き場所

| ログ | 場所 | 内容 |
|---|---|---|
| 変換の実行記録 | `/var/log/noim-run.log` | `sudo noim-run` ごとの変換ツールの出力(エラーメッセージを含む) |
| 起動時の配置の記録 | `/var/log/cloud-init-output.log` | ツールの取得・展開、`noim-run` の作成。配置に失敗したときの確認先 |

## よく使う確認

| 知りたいこと | コマンド |
|---|---|
| 直近の実行結果 | `sudo tail -n 50 /var/log/noim-run.log` |
| エラーが出ていないか | `sudo grep -i error /var/log/noim-run.log` |
| 配置が成功したか | `sudo tail -n 50 /var/log/cloud-init-output.log` |
| ツールが配置されているか | `ls -l /opt/noim` |
| 作業ディレクトリの出力 | `ls -l /var/noim/output` |

## 保存期間

- 保存期間の設定はありません。ファイルのままで、ローテーションもしません(量が小さいため)。
- EC2 が置き換わる(`cdk deploy` でコードを変えたとき)と、ログは消えます。必要なログは、先にコピーして残してください。

## 将来、CloudWatch Logs に送るとき

CloudWatch エージェントを入れて、上の2つのファイルを転送します。そのときに、Logs Insights のクエリと、ログの保存期間(例: 30日)を決めます。依頼してください。
