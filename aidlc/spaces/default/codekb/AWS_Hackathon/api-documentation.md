# API Documentation

HTTP / REST API は無い。このシステムの外部との接点は、CLI、入力 CSV、出力 CSV、EC2 のコマンド `noim-run` の 4 つ。可視化の設計で最も使うのは「出力 CSV の `data` の構造」(次の節)と「Reference Map」。

## REST APIs

なし。

## Internal APIs

### CLI: `noim_converter.py`

- 使い方: `python noim_converter.py <入力DIR> <出力DIR> [--expected <期待結果DIR>]`
- 終了コード: 0 成功 / 1 読み込み・変換・書き込み・比較の失敗(メッセージは stderr に `エラー: ...`)/ 2 引数エラー
- `--expected` を付けると、書き出した 8 ファイルを同名ファイルとバイト比較する。

### Python 関数(`noim/`)

| 関数 | 入力 → 出力 |
|---|---|
| `input_csv.load_inputs(dir)` | 入力 DIR → `(nodes, entities, services)`(`Row` のリスト) |
| `converters.convert_all(nodes, entities, services)` | → `{"nfds","fres","tpes","tls","pPorts","pds","pls","pStructs": [辞書,...]}` |
| `output_csv.write_outputs(result, dir)` | 結果 → 書き出したパスのリスト(順序は `RESOURCE_KINDS`) |
| `object_id.ObjectIdManager.next_id(kind)` | 種類 → `http://localhost:3000/<kind>/system_<8桁>` |

### EC2 のコマンド: `noim-run`

CDK の UserData が `/usr/local/bin/noim-run` に作る(`infra/noim_stack.py` の `NOIM_RUN_LINES`)。`sudo noim-run [引数]` で、`s3://<BUCKET>/input/` を `/var/noim/input` に取得 → `python3 /opt/noim/noim_converter.py /var/noim/input /var/noim/output "$@"` → `/var/noim/output` を `s3://<BUCKET>/output/` に戻す。ログは `/var/log/noim-run.log`。

## Data Models

### 入力 CSV(`入力ファイル/`)

- ファイル名: `t_mst_vxm_node_info_<12桁>.csv`、`t_mst_vxm_entity_info_<12桁>.csv`、`t_mst_vxm_service_<12桁>.csv`(サンプルは `202508051943`)。種類ごとに 12 桁が最大のファイルを使う。ディレクトリ直下にこの 3 種以外のファイルがあるとエラー。
- 形式: UTF-8(BOM は許容)、1 行目がヘッダ。必須列がそろっていないとエラー。名前が `_list` で終わる列は PostgreSQL の配列表記(`{a,b}`、`{"x y",z}`)で、文字列のリストとして読む。
- サンプルの件数: node 26、entity 463、service 51。

**ノード(`node_info`)の列(58 列)**

`ra_vxm_id`, `ra_job_id`, `ra_updated_at`, `ra_created_at`, `name`, `node_type`, `parent_node_group_id`, `parent_node_group_name`, `parent_node_id`, `parent_node_name`, `product_name`, `product_version`, `description`, `vendor_name`, `protocol`, `node_profile_id`, `node_profile_name`, `username`, `password`, `connection_type`, `primary_port`, `secondary_port`, `external_id`, `plug_in`, `telemetry_username`, `telemetry_password`, `telemetry_port`, `scope_id_list`, `scope_name_list`, `requests_per_time_period`, `admin_status`, `latitude`, `longitude`, `address`, `location`, `memo`, `oper_status`, `timezone`, `last_alarm_sync_at`, `primary_ipv4_address`, `primary_ipv6_address`, `secondary_ipv4_address`, `secondary_ipv6_address`, `unification_bldg_code`, `build_name`, `area_name`, `prefecture`, `floor_rack`, `host_name`, `neops_number`, `neops_serial_number`, `neops_traffic_url`, `neops_top_url`, `controller_vip_address`, `neops_ip_address`, `node_template`, `application_monitor_template`, `simple_control_template`

**エンティティ(`entity_info`)の列(33 列)**

`ra_vxm_id`, `ra_job_id`, `ra_updated_at`, `ra_created_at`, `name`, `description`, `memo`, `node_name`, `node_id`, `entity_profile_name`, `entity_profile_id`, `parent_entity_name`, `parent_entity_id`, `type`, `mac_address`, `ipv4_address`, `ipv6_address`, `admin_status`, `oper_status`, `mtu`, `speed`, `link_id`, `link_name`, `service_id`, `service_name`, `scope_id_list`, `scope_name_list`, `external_id`, `snmp_oid`, `snmp_index`, `snmp_type`, `snmp_type_id`, `created_by`

**サービス(`service`)の列(26 列)**

`ra_vxm_id`, `ra_job_id`, `ra_updated_at`, `ra_created_at`, `name`, `description`, `memo`, `service_profile_id`, `service_profile_name`, `admin_status`, `oper_status`, `protection_status`, `external_id`, `network_entity_id_list`, `network_entity_name_list`, `client_entity_id_list`, `client_entity_name_list`, `network_service_id_list`, `network_service_name_list`, `network_protection_service_id_list`, `network_protection_service_name_list`, `severity`, `priority`, `type`, `protection_type`, `service_status`

**入力どうしの参照(値による参照)**: `entity.node_id` → `node.ra_vxm_id`。`service.client_entity_id_list`(サンプルは全件 2 要素)→ `entity.ra_vxm_id`。`LSP詳細(BR)` の `service.network_service_id_list[0]` → 物理(BR) の `service.ra_vxm_id`。

**サンプルの値の分布(可視化の想定に効くもの)**: `node_type` は CE 12、SAS2 4、CER1 4、RR2 2、CR2 2、CER2 2。`unification_bldg_code` が空でないノードは 14(`build_name`、`host_name`、`primary_ipv4_address` も 14 件のみ値あり)。`name` は 26 件すべて一意。`latitude` / `longitude` は全件 `"0"`。`admin_status` は全件 `MONITORED`、`oper_status` は全件空。`service_profile_name` は 物理(BR) 24、VPN回線(BR) 19、LSP詳細(BR) 8。`entity.type` は 物理ポート 421、LAG 23、VPN回線 19。`username`、`password`、`telemetry_username`、`telemetry_password` は全件空。

### 出力 CSV(`resourceInfo_<種類>.csv` × 8)

- 形式: UTF-8(BOM なし)、CRLF、全フィールドを `"` で囲む。1 行目は `"objectId","data"`、以降 1 行 1 リソース。`data` は JSON 文字列(CSV としては `"` を `""` にする)。`ensure_ascii=False` なので日本語はそのまま入る。
- 種類(ファイル名の `<種類>`): `nfds`, `fres`, `tpes`, `tls`, `pPorts`, `pds`, `pls`, `pStructs`。
- `objectId`: `http://localhost:3000/<種類>/system_<8桁の連番>`(1 始まり、種類ごと、実行ごとに振り直し)。CE 用の PStruct だけ固定で `http://localhost:3000/pStructs/system_00000000`。
- `_resourceSpecification`: 種類ごとの固定の ID(対応するリソース定義の CSV は出力されない)。
  `lrSpecs/spec_00000001` = FRE、`spec_00000002` = TPE、`spec_00000003` = NFD、`spec_00000004` = TL、`prSpecs/spec_00000005` = PD、`spec_00000006` = PLink、`spec_00000007` = PPort、`spec_00000008` = PStruct(すべて `http://localhost:3000/` の下)。
- 共通: `resourceCapacity` は常に `[]`。物理系(PStruct、PD、PPort、PLink)は `nttLifecycleStatus: 1`、論理系(TPE、TL、FRE、NFD)は `lrStatus: 1`。

#### 各リソースの `data`

**PStruct(`pStructs`)** — 建物・拠点。1 建物 1 件(CE は「全国」1 件)。
`commonName`(建物名。CE は `全国`)、`objectId`、`_resourceSpecification`、`_place`(`[{ "ID": "1", "_localAddress": {"fullAddress": 名前}, "gridX": 緯度, "gridY": 経度 }]`。CE は `fullAddress` が `SYS=NOIM/Building=CE`、`gridX`/`gridY` が `"0"`)、`resourceCapacity`、`nttLifecycleStatus`、`nttManagementInfo`(常に `{}`)。
一意化のキー(CE は `全国`、`unification_bldg_code` があればその値、親グループが `BR網` ならノード名、それ以外はノード名を `・` で分けた 3 番目)。名前は、`unification_bldg_code` があれば `build_name`、`BR網` ならノード名、それ以外はノード名を `・` で分けた 3 番目(3 つに分かれなければノード名)。

**PD(`pds`)** — 装置。ノード 1 行につき 1 件。
`commonName`(= ノードの `name`)、`objectId`、`_resourceSpecification`、`resourceCapacity`、`_hardware`(`{"_physicalConnector": [{"_physicalPort": <PPort の objectId>}, ...]}`。そのノードのエンティティ分の PPort と、仮想 PPort)、`nttManagementInfo`(ノードの全列。キーは `node` + UpperCamelCase。例 `nodeRaVxmId`、`nodeName`、`nodeNodeType`、`nodeHostName`、`nodePrimaryIpv4Address`、`nodeBuildName`、`nodeAreaName`、`nodeAdminStatus`、`nodeOperStatus`。`nodeProfileName` だけ例外。配列列は JSON 配列。加えて `nttLadder`: CE は `1`、他は `null`)、`nttPhysicalStructure`(PStruct の objectId)、`nttPhysicalDeviceType`(常に `LOGICAL_CONNECT`)、`nttLifecycleStatus`。

**PPort(`pPorts`)** — ポート。エンティティ 1 行につき 1 件、さらに LAG 層の XC ごとに仮想 PPort 1 件。
`commonName`(エンティティの `name`)、`objectId`、`_place`(`fullAddress` 空、`gridX`/`gridY` 空)、`_resourceSpecification`、`resourceCapacity`、`vendorPortName`(= `commonName`)、`nttLifecycleStatus`、`nttManagementInfo`(エンティティの全列。キーは `entity` + UpperCamelCase。例 `entityRaVxmId`、`entityNodeId`、`entityNodeName`、`entityType`、`entityIpv4Address`、`entityServiceId`)。仮想 PPort は `commonName` と `vendorPortName` が `vPort` で、`nttManagementInfo` が `{}`。
PPort 自身は PD への objectId 参照を持たない。実ポートは `entityNodeId` で、仮想 PPort は PD 側の `_physicalConnector` と TPE 側の `_physicalPortRefList` からたどる。

**PLink(`pls`)** — 物理的な接続。`service_profile_name` が `物理(BR)` のサービス 1 行につき 1 件(サンプルは 24)。
`commonName`(サービスの `name`)、`objectId`、`_resourceSpecification`、`resourceCapacity`、`_physicalConnector`(`[{"_physicalPort": <PPort の objectId>}, ...]`。サービスの `client_entity_id_list` の各エンティティの PPort。通常 2 件)、`nttLifecycleStatus`、`nttManagementInfo`(サービスの全列。キーは `service` + UpperCamelCase。例 `serviceRaVxmId`、`serviceName`、`serviceClientEntityIdList`、`serviceClientEntityNameList`)。

**TL(`tls`)** — 論理装置層(LD 層)の接続。物理(BR) のサービス 1 行につき 1 件(PLink と同数・同順)。
`commonName`(サービスの `name`)、`objectId`、`_resourceSpecification`、`resourceCapacity`、`lrStatus`、`_aEndTpeRefList`(`[<TPE の objectId>]`)、`_zEndTpeRefList`(同)、`layerProtocolNameList`(`["logicalDevice"]`)、`transmissionParameterList`(`{"raVxmId": <サービスの ra_vxm_id>}`)。

**TPE(`tpes`)** — 端点。次の 4 種類が同じファイルに入る(ファイル内の並びは 1→2→3→4)。
共通のキー: `commonName`、`objectId`、`_resourceSpecification`、`resourceCapacity`、`lrStatus`、`_physicalPortRefList`、`_encapsulatedLtList`(要素 1 件: `layerProtocolNameList`、`ltType`、`transmissionParameterList`)、`tpeType`、`nttServerTpeRefList`。

| # | 種類 | `tpeType` | `layerProtocolNameList` / `ltType` | `commonName` | `_physicalPortRefList` | `transmissionParameterList` | `nttServerTpeRefList` |
|---|---|---|---|---|---|---|---|
| 1 | LD 層・物理(BR) 由来 | `PTP` | `["logicalDevice"]` / `TCP` | エンティティの `name` | そのエンティティの PPort(実ポート) | `ipAddress`(`ipv4_address`)、`deviceName`(`node_name`)、`raVxmId`(エンティティの `ra_vxm_id`) | `[]` |
| 2 | LAG 層 TCP | `CTP` | `["lag"]` / `TCP` | LSP詳細(BR) のサービスの `name` | `[]` | 3 項目とも空文字 | `[#4 の TPE]` |
| 3 | LAG 層 CP | `CTP` | `["lag"]` / `CP` | #1 の TPE の `commonName` | `[]` | #1 の 3 項目のコピー | `[#1 の TPE]` |
| 4 | LD 層・仮想 PPort 相当 | `PTP` | `["logicalDevice"]` / `TCP`(#2 から写す) | #2 の `commonName` | `[仮想 PPort]` | 3 項目とも空文字 | `[]` |

#1 と #4 は `ltType`、`layerProtocolNameList` が同じ。区別は `transmissionParameterList.deviceName` が空かどうか(#4 は空)。

**FRE(`fres`)** — 論理的な接続(LAG 層)。`nttFreType` で 3 種類に分かれる。ファイル内の並びは NC → LC → XC。
共通のキー: `commonName`、`objectId`、`_resourceSpecification`、`resourceCapacity`、`lrStatus`、`_endPointList`(`[{"_tpeIdRef": <TPE の objectId>}, ...]`)、`_supportingFreRefList`、`freType`(`POINT_TO_POINT`)、`layerProtocolName`(`lag`)、`transmissionParameterList`(`{"raVxmId": ...}`)、`nttFreType`、`nttServerConnectionRefList`。

| `nttFreType` | 作られる単位 | `_endPointList` | `_supportingFreRefList` | `nttServerConnectionRefList` | `raVxmId` の値 |
|---|---|---|---|---|---|
| `NC` | A 側ノードと Z 側ノードの組ごとに 1 件 | [A の TCP、Z の TCP](#2) | [A 側の XC、Z 側の XC] | `[]` | LSP詳細(BR) のサービス(最初の 1 件)の `ra_vxm_id` |
| `LC` | LSP詳細(BR) のサービス 1 行につき 1 件(サンプルは 8) | [A の CP、Z の CP](#3) | `[]` | `[TL の objectId]` | そのサービスの `ra_vxm_id` |
| `XC` | ノードごと(A 側・Z 側それぞれの対応表で 1 件ずつ) | [そのノードの TCP、CP(複数)] | `[]` | `[]` | **ノードの `ra_vxm_id`**(= PD の `nodeRaVxmId`) |

**NFD(`nfds`)** — XC ごとに 1 件(XC と同数・同順)。
`commonName`(XC の `commonName`)、`objectId`、`_resourceSpecification`、`resourceCapacity`、`lrStatus`、`_edgeTpeRefList`(XC の端点 TPE の `nttServerTpeRefList` を並べたもの。つまり #4 と #1 の TPE)、`_layerProtocolParameters`(`[{"layerProtocolName": "logicalDevice", "transmissionParameterList": {}}]`)、`layerProtocolNameList`(`["logicalDevice"]`)、`nfdRole`(`ROUTING_AREA`)、`nttSupportableClientLayers`(`["lag"]`)。
NFD は XC への参照を持たない。対応づけは、作成順(同じ位置)か `_edgeTpeRefList` 内の #1 の TPE の `deviceName` でする。

## Reference Map

どの項目がどのリソースを指すか。`objectId` 参照は左側の項目の値が右側リソースの `objectId` と一致する。値参照は入力の ID(`ra_vxm_id`)や名前の一致。

| 起点(リソース . 項目) | 終点 | 種別 |
|---|---|---|
| PD . `nttPhysicalStructure` | PStruct の `objectId` | objectId |
| PD . `_hardware._physicalConnector[]._physicalPort` | PPort の `objectId` | objectId |
| PPort . `nttManagementInfo.entityNodeId` | PD の `nttManagementInfo.nodeRaVxmId` | 値(実ポートのみ) |
| PLink . `_physicalConnector[]._physicalPort` | PPort の `objectId` | objectId |
| PLink . `nttManagementInfo.serviceRaVxmId` | TL の `transmissionParameterList.raVxmId` | 値(同じサービス。`commonName` も同じ) |
| TL . `_aEndTpeRefList[]`、`_zEndTpeRefList[]` | TPE #1 の `objectId` | objectId |
| TPE #1 . `_physicalPortRefList[]` | PPort の `objectId` | objectId |
| TPE #1 . `transmissionParameterList.deviceName` | PD の `commonName`(= `nttManagementInfo.nodeName`) | 値 |
| TPE #1 . `transmissionParameterList.raVxmId` | PPort の `nttManagementInfo.entityRaVxmId` | 値 |
| TPE #2(TCP) . `nttServerTpeRefList[]` | TPE #4 の `objectId` | objectId |
| TPE #3(CP) . `nttServerTpeRefList[]` | TPE #1 の `objectId` | objectId |
| TPE #4 . `_physicalPortRefList[]` | 仮想 PPort の `objectId` | objectId |
| FRE . `_endPointList[]._tpeIdRef` | TPE #2 / #3 の `objectId` | objectId |
| FRE(NC) . `_supportingFreRefList[]` | FRE(XC)の `objectId` | objectId |
| FRE(LC) . `nttServerConnectionRefList[]` | TL の `objectId` | objectId |
| FRE(XC) . `transmissionParameterList.raVxmId` | PD の `nttManagementInfo.nodeRaVxmId` | 値 |
| NFD . `_edgeTpeRefList[]` | TPE #1 / #4 の `objectId` | objectId |
| すべて . `_resourceSpecification` | 固定の spec ID(参照先ファイルなし) | 固定値 |

## 装置と装置間の接続に当たるもの

- **装置**: PD(`resourceInfo_pds.csv`)。名前は `commonName`、安定した識別は `nttManagementInfo.nodeRaVxmId`。拠点は PD . `nttPhysicalStructure` → PStruct(建物。サンプルでは座標が `"0"` なので位置は使えない)。
- **物理的な装置間の接続**: PLink(`pls`)。PLink . `_physicalConnector[]` の 2 つの PPort を、PPort . `nttManagementInfo.entityNodeId` で PD に結ぶ。TL(`tls`)は同じサービスの LD 層での表現で、`_aEndTpeRefList` / `_zEndTpeRefList` の TPE #1 の `deviceName` から、PD を経由せずに A 側・Z 側の装置名が取れる。PLink と TL は 1 対 1(同じ順序、同じ `commonName`)。
- **論理的な装置間の接続(LSP)**: FRE の NC(ノードの組ごと)と LC(LSP詳細(BR) のサービスごと)。NC . `_supportingFreRefList` の XC の `raVxmId` が両端のノード(= PD . `nodeRaVxmId`)。LC は `nttServerConnectionRefList` で TL(物理の接続)に結ばれる。
- **接続にならないもの**: `VPN回線(BR)` のサービス(サンプルで 19 件)は、PLink、TL、FRE のどれにもならない(対応する PPort だけ作られる)。

## Alarm matching keys(アラームファイルとの突き合わせの候補)

アラームファイルの仕様はこのリポジトリに無い。装置側で使える項目は次のとおり。

| 項目(PD . `...`) | 実際に値がある範囲(サンプル) | 備考 |
|---|---|---|
| `commonName` / `nttManagementInfo.nodeName` | 26 / 26(一意) | 入力の `name`。日本語を含む |
| `nttManagementInfo.nodeRaVxmId` | 26 / 26 | UUID。入力 DB 側の ID |
| `nttManagementInfo.nodeHostName` | 14 / 26 | CE 12 件は空 |
| `nttManagementInfo.nodePrimaryIpv4Address` | 14 / 26 | CE 12 件は空 |
| `objectId` | 26 / 26 | **使わない**。実行ごとに採番し直される |

## 出力に含まれる機微な項目

`nttManagementInfo` はノードの `username`、`password`、`telemetry_username`、`telemetry_password` も写す(`nodeUsername`、`nodePassword`、`nodeTelemetryUsername`、`nodeTelemetryPassword`)。サンプルは空だが、実データでは値が入りうる。可視化ではこれらを読み込んだり表示したりしないこと。
