# Requirements Analysis Questions

依頼: 「NOIM変換ツールの出力(装置と装置間の接続情報)を、AWS上でビジュアライズ化したい。また、S3に置いたアラームファイル(JSON)を読み込み、アラームが発生している装置を見える化したい。」

すでに分かっていること(質問しません):
- 装置は出力の `resourceInfo_pds.csv`(PD、26件)、装置間の接続は `resourceInfo_pls.csv`(PLink)・`resourceInfo_tls.csv`(TL)・`resourceInfo_fres.csv`(FRE。LSPの論理接続)で表せます。
- アラームJSONの形式は `アラーム/alarm_spec.md` のとおりです(`alarmId`、`alarmName`、`alarmLevel` 1〜3、`targetDeviceId`、`occurredAt`、`status` は `active` または `recovered`)。この仕様書とサンプルは、実装に合わせて編集して構いません(ご指示のとおり)。
- 出力の `data` には、入力の `username`、`password`、`telemetry_*` の列も写ります。**これらは画面にも、ブラウザに送るデータにも含めません。**
- 変換ツールの出力の `objectId` は実行ごとに振り直されるので、アラームとの突き合わせには使えません。
- EC2と変換ツールは、そのまま使います。デプロイは、これまでどおりあなたがCloudShellでCDKを実行します。

## Q1. [プロダクトマネージャー] アラームの装置を、構成図のどの装置に結びつけるか

アラームの `targetDeviceId`(サンプルは `DEV-001` の形)を、構成図の装置に結びつける方法です。NOIM側の装置は、名前(例: `関西・アバンザCER1-A`、26件すべて一意)で区別できます。ホスト名とIPアドレスは、26件中14件にしか値がありません。

A. `targetDeviceId` に、NOIMの装置名(`commonName`)と同じ文字列を入れる運用にする。`alarm_sample.json` と `alarm_spec.md` を、この形に書き換える (Recommended)
B. 対応表(例: `DEV-001` → 装置名)を別のJSONとしてS3に置き、画面がそれを読んで結びつける
C. ホスト名またはIPアドレスで結びつける(値のある14件の装置だけが対象になる)
X. Other (please specify)

[Answer]: A

## Q2. [プロダクトマネージャー] 構成図に出す接続の範囲

サンプルでは、物理的な接続(PLink と TL)が24本、LSPの論理的な接続(FRE の NC)が8本あります。`VPN回線(BR)` の19サービスは、変換で接続リソースになっていないため、図には出せません。

A. 物理接続とLSPの論理接続の両方を出し、線の見た目(実線と点線など)で区別する (Recommended)
B. 物理接続だけを出す
C. LSPの論理接続だけを出す
X. Other (please specify)

[Answer]: A

## Q3. [デザイナー] 構成図の見せ方

A. ブラウザで見るWebページ。装置を点、接続を線で描く(自動で配置し、拡大・縮小、装置のドラッグ、装置をクリックして詳細の表示ができる) (Recommended)
B. 画像(SVGなど)のファイルを作ってS3に置く(見るだけ。操作はできない)
C. Amazon QuickSight などのBIツールを使う
X. Other (please specify)

[Answer]: A

## Q4. [AWSプラットフォームエンジニア] 画面を誰が、どう見られるようにするか

構成情報とアラームは社内のネットワーク情報なので、誰でも見られる状態にはしません。

A. CloudFront + S3 で配信し、Basic認証(共有のIDとパスワード)をかける。ID・パスワードはデプロイ時にあなたが指定する (Recommended)
B. CloudFront + S3 で配信し、AWS WAFで、許可したIPアドレスだけに限定する(WAFの費用が月数ドルかかる)
C. Amazon Cognito でユーザー認証をする(ユーザー管理の手間が増える)
X. Other (please specify)

[Answer]: X. ローカルの本PC上で見れればよい

## Q5. [運用担当] 最新のデータを、いつ画面に反映するか

変換結果とアラームのファイルは、S3に置き直されるたびに変わります。

A. 画面を開いたとき、および「更新」ボタンを押したときに、その都度S3の最新のファイルを読んで描く。裏側は、Lambda(関数)が出力CSVとアラームJSONを読み、認証情報の列を取り除いて、画面用のデータに整える (Recommended)
B. S3にファイルが置かれたときに、Lambdaが自動で整形して保存しておき、画面はそれを読むだけにする(画面の表示は速いが、仕組みが増える)
X. Other (please specify)

[Answer]: A

## Q6. [プロダクトマネージャー] アラームの表示ルール

アラームには重大度(1 軽度、2 中度、3 重度)と状態(`active` 発生中、`recovered` 復旧済み)があります。同じ装置に複数のアラームが付くことがあります。

A. `active` のアラームがある装置だけを強調する。色は重大度で分ける(1 黄、2 橙、3 赤)。装置に複数のアラームがあるときは、最も重い重大度の色にする。装置をクリックすると、その装置のアラーム一覧(名前、重大度、発生日時、状態)を見られる。`recovered` は強調せず、一覧にだけ薄く表示する (Recommended)
B. `recovered` も含めてすべて強調する(復旧済みは灰色)
X. Other (please specify)

[Answer]: A
