# Monitoring Design: NOIM変換ツール

PoC のため最小限にする。継続的に動くサービスではなく、人が実行するバッチなので、通知を出す常時の監視は置かない。詳細な監視の設定は、計画上の Observability Setup で扱う。

## Metrics & KPIs

| Metric | Source | Threshold | Why it matters |
|---|---|---|---|
| `StatusCheckFailed` | CloudWatch(EC2 標準メトリクス) | 0 以外が 2 回連続 | EC2 が壊れて SSM で入れない状態に気づく |
| `CPUUtilization` | CloudWatch(EC2 標準メトリクス) | 参考値として見る(しきい値は設定しない) | t3.micro で足りているかの目安。連続で高いなら t3.small にする |
| 変換の終了ステータス | `noim-run` の出力 | 0 以外は失敗 | 変換の成否を人が見て判断する |

## Alerts

| Alert | Condition | Severity | Routes to |
|---|---|---|---|
| EC2 ステータスチェック失敗 | `StatusCheckFailed` が 5 分間 連続で 0 超 | 低(チケット相当) | CloudWatch アラームの状態を見る。通知先(SNS)は作らない。必要になったら指示 |

## SLIs / SLOs

該当なし。常時のサービスではなく、可用性の目標は設けない。

## Logs & Tracing

| Item | Where | Notes |
|---|---|---|
| 起動時の配置の記録 | EC2 の `/var/log/cloud-init-output.log` | 配置に失敗したときの確認先。SSM で入って見る |
| 変換の実行記録 | `noim-run` の標準出力 + `/var/log/noim-run.log` | 標準エラー出力に出るエラーメッセージ(ファイル名つき)も同じファイルに残す |
| CloudWatch Logs への転送 | 作らない | CloudWatch エージェントは入れない(PoC) |
| トレース、ダッシュボード | 作らない | 該当しない |
