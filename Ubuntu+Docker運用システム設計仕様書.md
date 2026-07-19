# Ubuntu + Docker 運用システム 設計仕様書(まとめ)

## 1. 前提条件

- **対象サービス**: OpenProject, Growi, Nexus (Docker Composeで運用)
- **利用規模**: 開発環境、約50人規模
- **可用性要件**:
  - 夜間・休日の停止は許容(緩め)
  - 業務時間 08:50-17:20 はサービス継続が必要
- **構成**: 各サービスにつき2台のVMを用意し、故障時に切り替える運用
  - VM-A: 稼働系(Primary)
  - VM-B: 待機系(Standby)
- **重要データ**: OpenProjectのプロジェクトデータ、Growiのwiki、Nexusのアーティファクトは、いずれも重要データとして扱う
- **リソース制約**: 複数世代のバックアップ保持、物理的なオフサイト(別拠点)保管を行うだけのリソースはない

## 2. 可用性・切替方式

- **リバースプロキシ**: 可用性のボトルネックになるため不採用
- **DNS Aレコード切替方式を採用**
  ```
  serviceA.example.com        -> 10.10.10.10 (稼働系)
  serviceA-backup.example.com -> 10.10.10.11 (待機系、常時疎通確認用)
  
  障害時: serviceA.example.com のAレコードを 10.10.10.11 に切替
  ```

- **DNS切替の実行スコープ**: 別チームの管轄のため、当システムの設計スコープ外

- **当システム側の責務**:
  - 稼働系・待機系の死活監視
  - 障害検知時のアラート発報(DNS担当への連絡材料を含む)
  - 待機系のデータ鮮度・整合性の可視化
  - TLS証明書の両系配布
  - 切替後のアプリ動作確認

- **切替方式**: 半自動(監視は自動、切替の実行判断は人手)
  - **理由**: 定期(日次)レプリケーションのためタイムラグがあり、完全自動切替は不整合データへの誤切替リスクがあるため

## 3. データ同期方式

### 同期方式
日次データダンプ + SCP転送 + 待機系でリストア

※ 将来的にはDBネイティブなストリーミングレプリケーション/レプリカセット方式への移行を予定

### VM-A(稼働系)のフロー

1. DBダンプ / ファイルアーカイブ作成
2. 直近1世代のみ保持(世代管理は最小限、リソース制約のため)
3. SCPで待機系(VM-B)へ転送
4. SSH経由でVM-Bのリストアスクリプトをキック(ダンプ・転送・リストアのタイミングずれを防止)
5. 基準値(baseline)を更新
6. Uptime Kumaへ成功Push通知

### VM-B(待機系)のフロー

- リストアスクリプトが受信データを元に洗い替え
- リストア完了ログを記録

### サービスごとの同期特性

| サービス | 方式 |
|---------|------|
| OpenProject | PostgreSQL dump (pg_dump/pg_restore) + 添付ファイルtar |
| Growi | MongoDB dump (mongodump/mongorestore) + 添付ファイルtar |
| Nexus | rsync --delete (差分転送、ネイティブレプリケーション機構なし) |

### RPOの認識合わせ

- 日次ダンプのため、最大で前日実行時点までのデータロスが発生しうる
  - 例: 17:20に稼働系故障 → 前日02:00時点まで巻き戻る
- 必要であれば業務時間終了直後にダンプを追加しRPO短縮可能(将来検討事項)

### SCP/SSH用ユーザーのセキュリティ

- **現状**: 通常のSSH鍵認証のみ(コマンド制限等は未実施)
- **将来対応(保留事項)**:
  - `authorized_keys` への `command=` 制限
  - `no-port-forwarding` 等の付与
  - `restore-wrapper.sh` による実行コマンド限定

## 4. Docker Compose 構成方針

- `compose.yml` は稼働系・待機系で完全に同一のファイルを使用
- 稼働系/待機系の差分は `.env` ファイルのみで表現
- 構造的な差分(マウントパス等)がある場合のみ `compose.override.yml` を使用(自動マージされるため -f不要)

### アプリ層の起動制御

Docker Compose の `profiles` 機能を使用し、`.env` の `COMPOSE_PROFILES` で制御

```
VM-A(稼働系): COMPOSE_PROFILES=active  → アプリ起動
VM-B(待機系): COMPOSE_PROFILES=        → アプリ停止(DBのみ起動、リストアを受付)
```

### ディレクトリ構成(共通パターン)

```
/opt/<service>/
├── compose.yml            # 稼働系・待機系で完全同一。Git管理
├── compose.override.yml   # 必要な場合のみ
├── .env                   # VMごとに異なる値(役割・FQDN等)
├── secrets/
└── scripts/               # dump-and-restore.sh, restore.sh等
```

### Growiのレプリカセット関連設定

`rs.initiate`等は今回の「日次dump方式」では不要
(将来のレプリケーション移行時に再検討)

## 5. 監視・アラート設計

### 監視ツール

**Uptime Kuma を採用**

- Zabbixは導入済みだったが、規模に対してオーバースペック、運用の手間が大きいため不採用と判断

### Uptime KumaのDB

PostgreSQLを使用
- OpenProjectと同一エンジンのため保守知見を共用できる
- `compose.yml`上で `uptime-kuma-db(postgres)` + `uptime-kuma` の2サービス構成とする

### データ保持期間

100日(3か月の傾向分析 + 予備)
- Settings → Monitor History → Data Retention Period: 100

### 監視対象と方式

| 監視対象 | 方式 |
|---------|------|
| VM-A / VM-B 死活 | ICMP(ping)監視 |
| 稼働系アプリ(HTTP) | HTTP(s)監視 |
| 待機系DB(ポート) | TCP監視 |
| dump/転送/リストアジョブ | Push監視(heartbeat) |
| サーバーリソース(CPU/MEM/DISK) | Push監視のping値 |
| 脆弱性スキャン | Push監視(heartbeat) |
| マルウェア検知(mdatp) | Push監視(heartbeat、リアルタイム監視は常駐のためPush対象外) |

#### dump/転送/リストアジョブの監視

- スクリプトの最終行でのみ成功通知を送る
- 途中で失敗すればPushが届かず異常検知できる

#### サーバーリソースの監視

- Push監視のping値(応答速度ms)に使用率(%)を代入して送信する方式(仕様書ベース)
- 例: 10ms = 使用率10% と読み替えて運用

#### 収集スクリプト概要(全VM共通、1分毎にcron実行)

```bash
- mpstat  → CPU使用率
- free    → メモリ使用率
- df      → ディスク使用率
- 上記をUptime KumaのPush URLへ curl送信
```

- `msg` にホスト名を含め、どのVMの値か識別可能にする
- 閾値超過時の自動アラート(`status=down`分岐)は将来対応として見送り、現状は常時 `status=up` で送信

### モニター命名規則

```
[VM-A][OpenProject] CPU / MEM / DISK
[VM-B][OpenProject-Standby] CPU / MEM / DISK
[VM-A][Growi] CPU / MEM / DISK
[VM-A][Nexus] CPU / MEM / DISK  等
[監視サーバ][Trivy] Vulnerability Scan
[VM-A][mdatp] Malware Scan
[VM-B][mdatp] Malware Scan
```

### アラート通知先

- Slack等(社内ツールのWebhook)
- DNS担当への連絡経路も通知フローに含める

### 監視サーバーの配置

VM-A/VM-Bとは別の監視専用VM(またはは既存管理サーバー)に設置
- 稼働系障害時に監視も道連れにならないようにするため

## 6. 脆弱性点検設計(Trivy)

### 方針

- **ツール**: Trivy (コンテナイメージ、OS、アプリケーション依存関係の脆弱性スキャン)
- **実行環境**: 監視専用VM(第5章参照)で集中管理。ただし、実際のスキャン処理自体は各スキャン対象VM上で実行し(下記参照)、監視専用VMはSSH経由での実行指示と結果収集を担う
- **スキャン対象**: コンテナイメージ、稼働系/待機系 OS、アプリケーション依存関係
- **実行頻度**: 日次(夜間 02:00、dump/restore処理と前後しない時間帯)

### 採用理由

- Docker Compose 環境に最適化されている
- 複数のスキャン対象(イメージ/OS/依存関係)を1ツールでカバー可能
- 既存の Uptime Kuma 監視体系と統合容易
- 開発環境規模に対して運用負荷が軽い

### 前提構成:Trivyのインストール先

`trivy image`はDockerデーモンへのアクセスが必須であり、`trivy filesystem`は対象ホストのファイルシステムに直接アクセスする必要があるため、いずれもスキャン処理自体は**対象VM上で実行**する。したがって、**Trivyは監視専用VMだけでなく、各スキャン対象VM(VM-A/VM-B)にもインストールする**。監視専用VMは、SSH経由でスキャン対象VM上のTrivyを実行し、結果(JSON)を受け取る役割を担う。

### スキャン対象と優先度

| 優先度 | 対象 | 理由 | スキャン対象 |
|--------|------|------|------------|
| 🔴 高 | OpenProject(VM-A) | 重要データ保有(プロジェクト) | コンテナイメージ + OS |
| 🔴 高 | Growi(VM-A) | 重要データ保有(wiki) | コンテナイメージ + OS |
| 🟠 中 | Nexus(VM-A) | アーティファクト保管 | コンテナイメージ + OS |
| 🟠 中 | VM-A OS(Ubuntu) | 本体 OS 脆弱性 | Filesystem スキャン |
| 🟡 低 | VM-B OS(Ubuntu) | 待機系(起動優先度低) | Filesystem スキャン(週1回) |

### 事前準備:スキャン対象VMのディスク容量

Trivyは初回実行時に以下のデータをダウンロードするため、スキャン対象VM側に十分な空き容量を確保する。

| データ | サイズ目安 |
|--------|-----------|
| 脆弱性DB(trivy-db) | 約150〜300MB |
| Java依存関係DB(trivy-java-db) | 約400〜600MB |
| Dockerイメージ本体 | 対象イメージによる |

事前に `df -h` および `df -i`(inode使用率)を確認しておく。Javaアプリケーションを含まない対象(OSスキャンのみ等)では、以下のオプションでJava DBダウンロードをスキップし、容量・速度の両面で負荷を下げる。

```bash
--pkg-types os          # OSパッケージのみ対象、言語別DB(Java等)は不要に
--scanners vuln         # 脆弱性検出のみ(Secret/Misconfigurationスキャンを除外、高速化)
```

### OSスキャン実行時の注意

`trivy filesystem /` はデフォルトで Secret scanning(全ファイルの中身をパターンマッチで走査)も有効なため、ルートファイルシステム全体が対象だと処理に時間がかかる。運用スクリプトでは `--scanners vuln` を明示指定し、脆弱性検出のみに絞ることで高速化する。

### スクリプト配置と実行

#### ディレクトリ構成

```
/opt/monitoring/trivy/
├── vulnerability-scan.sh    # 主スクリプト
├── trivy-config.yaml        # Trivy設定ファイル
├── reports/                 # スキャン結果レポート保管
│   ├── openproject-image.json
│   ├── openproject-os.json
│   ├── growi-image.json
│   ├── growi-os.json
│   ├── nexus-image.json
│   └── vm-a-fs.json
└── baseline/                # 基準値管理(脆弱性数の前回値)
    └── baseline-severity.txt
```

#### 所有者・権限

スキャン結果JSONには対象システムの脆弱性情報(攻撃者にとって有用な情報)が含まれるため、他ユーザーからの読み取りを制限する。

```bash
sudo mkdir -p /opt/monitoring/trivy/{reports,baseline}
sudo chown -R <実行ユーザー>:<実行ユーザー> /opt/monitoring/trivy
sudo chmod 750 /opt/monitoring/trivy /opt/monitoring/trivy/reports /opt/monitoring/trivy/baseline
sudo chmod 750 /opt/monitoring/trivy/vulnerability-scan.sh   # Slack Webhook URL等の秘匿情報を含むため
sudo chmod 640 /opt/monitoring/trivy/trivy-config.yaml
sudo chmod 640 /opt/monitoring/trivy/reports/*.json
```

- cronの実行ユーザーと`/opt/monitoring`の所有者は一致させる(root cronならroot所有、一般ユーザーcronならそのユーザー所有)
- 本番運用では、`ubuntu`等の汎用ユーザーではなく専用サービスアカウント(例: `trivyscan`)の作成を推奨(最小権限の原則)

#### cron 設定(監視専用VM)

```bash
# 毎日 02:00 に脆弱性スキャン実行
0 2 * * * /opt/monitoring/trivy/vulnerability-scan.sh >> /var/log/trivy-scan.log 2>&1
```

#### vulnerability-scan.sh の概要

```bash
#!/bin/bash

# 監視対象のコンテナイメージ、OS のスキャン(SSH経由でスキャン対象VM上のTrivyをリモート実行)
# 前回比較で新規/増加脆弱性を検知

# 1. コンテナイメージスキャン(VM-A上でリモート実行)
ssh vm-a "trivy image openproject:latest --severity HIGH,CRITICAL --format json --skip-version-check" \
  > /opt/monitoring/trivy/reports/openproject-image.json

ssh vm-a "trivy image growi:latest --severity HIGH,CRITICAL --format json --skip-version-check" \
  > /opt/monitoring/trivy/reports/growi-image.json

ssh vm-a "trivy image nexus:latest --severity HIGH,CRITICAL --format json --skip-version-check" \
  > /opt/monitoring/trivy/reports/nexus-image.json

# 2. OS(ファイルシステム)スキャン(VM-A/VM-Bへリモート実行)
# --scanners vuln で Secret scanning を除外し高速化
ssh vm-a "trivy filesystem / --severity HIGH,CRITICAL --format json --skip-version-check --scanners vuln" \
  > /opt/monitoring/trivy/reports/vm-a-fs.json

# 3. 脆弱性カウント集計(jq -s でスラープし、複数JSONファイルを安全に結合)
CRITICAL_COUNT=$(jq -s '[.[].Results[].Vulnerabilities[]?] | map(select(.Severity=="CRITICAL")) | length' \
  /opt/monitoring/trivy/reports/*.json)

HIGH_COUNT=$(jq -s '[.[].Results[].Vulnerabilities[]?] | map(select(.Severity=="HIGH")) | length' \
  /opt/monitoring/trivy/reports/*.json)

# 4. 基準値との比較(初回実行時は基準値設定、以降は前回値と比較)
if [ -f /opt/monitoring/trivy/baseline/baseline-severity.txt ]; then
  PREV_CRITICAL=$(grep "CRITICAL" /opt/monitoring/trivy/baseline/baseline-severity.txt | awk '{print $2}')
  PREV_HIGH=$(grep "HIGH" /opt/monitoring/trivy/baseline/baseline-severity.txt | awk '{print $2}')

  # 新規脆弱性 or 増加検知時は異常判定
  if [ "$CRITICAL_COUNT" -gt "$PREV_CRITICAL" ] || [ "$HIGH_COUNT" -gt "$PREV_HIGH" ]; then
    STATUS="down"
    MSG="[Trivy Alert] CRITICAL: $CRITICAL_COUNT(prev:$PREV_CRITICAL) HIGH: $HIGH_COUNT(prev:$PREV_HIGH)"
  else
    STATUS="up"
    MSG="[Trivy OK] CRITICAL: $CRITICAL_COUNT HIGH: $HIGH_COUNT"
  fi
else
  # 初回実行: 基準値設定
  STATUS="up"
  MSG="[Trivy Baseline] CRITICAL: $CRITICAL_COUNT HIGH: $HIGH_COUNT (初回実行)"
fi

# 5. 基準値更新
cat > /opt/monitoring/trivy/baseline/baseline-severity.txt << EOF
CRITICAL $CRITICAL_COUNT
HIGH $HIGH_COUNT
EOF

# 6. Uptime Kuma へ Push 通知
KUMA_URL="http://uptime-kuma:3001/api/push/trivy-scan"
curl -X POST "$KUMA_URL?status=$STATUS&msg=$MSG&ping=$(date +%s)"

# 7. 高リスク脆弱性検知時、Slack へも通知(オプション)
if [ "$STATUS" = "down" ]; then
  SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."
  curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"⚠️ Trivy脆弱性アラート: $MSG\"}" \
    "$SLACK_WEBHOOK_URL"
fi
```

### Trivy 設定ファイル(trivy-config.yaml)

```yaml
severity:
  - HIGH
  - CRITICAL

# スキャン対象(デフォルト)
scanners:
  - vuln        # 脆弱性スキャン
  - misconfig   # 設定ファイル問題

# DBキャッシュ(オプション)
cache:
  dir: /opt/monitoring/trivy/.trivy-cache

# レポート形式
format: json

# 出力詳細度
debug: false
```

### レポート保管と分析

#### レポートの管理

- **保管先**: `/opt/monitoring/trivy/reports/`
- **保管期間**: 90日(Trivy スキャン結果の傾向分析用)
- **命名規則**: `{service-name}-{scan-type}-{YYYY-MM-DD}.json`
  - 例: `openproject-image-2026-01-15.json`

#### 分析と対応

1. **CRITICAL 脆弱性**: 即日対応(パッチ適用またはリスク承認)
2. **HIGH 脆弱性**: 週内対応(スプリント内で修正)
3. **MEDIUM 以下**: 月1回の定期レビュー

#### FixedVersionの有無による仕分け

CRITICAL/HIGHの件数だけでは対応の緊急性を正しく判断できない。同じSeverityでも「パッチ適用で解消可能なもの」と「上流で未修正のもの」が混在するため、以下の仕分けを対応フローに組み込む。

```bash
# パッチ適用可能 / 未対応の仕分け
jq '[.Results[].Vulnerabilities[] | select(.Severity=="CRITICAL")] |
    group_by(.FixedVersion != null) |
    map({has_fix: (.[0].FixedVersion != null), count: length})' \
  <report.json>
```

| 分類 | 対応方針 |
|------|---------|
| FixedVersionあり | パッチ適用(apt upgrade等)で解消可能 → 即日対応 |
| FixedVersionなし | 上流未対応 → リスク承認 → スキップリスト登録し、定期的に再確認 |

### 古いカーネルパッケージの取り扱い

`apt upgrade` および再起動を実施しても、直前バージョンのカーネル関連パッケージ(`linux-image`, `linux-headers`, `linux-modules`, `linux-tools` 等)が `apt autoremove` で自動削除されず、Trivyの脆弱性検出に残り続けるケースがある。AWS(linux-aws系)カーネルパッケージは自動削除対象としてマークされない場合があり、Ubuntu標準カーネルより発生しやすい。

**運用ルール**:
- カーネル更新・再起動後、新カーネルの安定稼働を確認したうえで、明示的に旧カーネル関連パッケージを `apt purge` する運用手順を定期メンテナンスに組み込む
  ```bash
  # 例:現在のカーネルバージョンを確認した上で、旧バージョンの関連パッケージを個別に指定してpurge
  uname -r
  dpkg -l | grep linux- | grep <旧バージョン番号>
  sudo apt purge -y <該当パッケージ一覧>
  ```
- もしくは、旧カーネル起因の検出をスキップリストとして許容し、実害がないことを定期レビューで確認する運用とする

### 脆弱性への対応フロー

```
Trivy スキャン実行(日次 02:00)
  ↓
CRITICAL/HIGH 脆弱性 検出?
  ├─ YES → Uptime Kuma: status=down
  │         ↓
  │         Slack 通知(セキュリティ/DevOps)
  │         ↓
  │         FixedVersion 有無を確認
  │         ├─ 有 → パッチ適用 → イメージ/OS 更新 → 再スキャン
  │         └─ 無 → リスク承認 → チケット化 → スキップリスト登録
  │
  └─ NO → Uptime Kuma: status=up (正常)
          基準値更新
```

### 検出できる脆弱性 / できない脅威

#### ✅ 検出できるもの

- コンテナベースイメージの既知脆弱性(CVE)
- OS パッケージ(apt, yum等)の脆弱性
- Python, Node.js, Java 等アプリ依存パッケージの脆弱性
- Docker/Kubernetes 設定ミス

#### ❌ 検出できないもの

- ロジック的な脆弱性(SQLインジェクション等)
- ゼロデイ脆弱性(未公開 CVE)
- SCP/SSH 認証情報漏洩の設定ミス(現状 command= 制限なし)
- アプリケーション側の認可ロジックの不備

### 将来の拡張(保留事項)

- [ ] Trivy の脆弱性DBを定期更新(毎日 03:00 実行)
- [ ] 脆弱性スキップリスト(false positive 対応)の管理ポリシー
- [ ] CI/CD パイプライン(GitHub Actions等)への統合
- [ ] より詳細なレポート生成(HTML形式で月次レビュー用)
- [ ] 旧カーネルパッケージの定期クリーンアップ手順の自動化検討
- [ ] 監視専用VM・スキャン対象VM双方のTrivyバージョン統一/更新管理手順の整備

## 7. マルウェア対策設計(Microsoft Defender for Endpoint / mdatp)

### 方針

- **ツール**: Microsoft Defender for Endpoint on Linux(パッケージ名 `mdatp`)
- **対象OS**: Ubuntu 24.04 LTS(VM-A / VM-B)
- **役割**: Trivyが担う「脆弱性(CVE)スキャン」とは明確に分離し、mdatpは「マルウェア/ウイルスの検出・駆除」を担当
- **管理**: Microsoft Defenderポータル(M365 Defender Premium)でのクラウド一元管理

### Trivyとの役割分担

| ツール | 役割 | 検出対象 | 実行環境 |
|--------|------|---------|---------|
| Trivy | 脆弱性(CVE)スキャン | コンテナイメージ/OS/依存関係の既知脆弱性 | 監視専用VM(集中管理) |
| mdatp | マルウェア/ウイルス検出 | 侵入したマルウェア・不審なファイル操作 | VM-A / VM-B 各ホスト常駐 |

→ 両者は競合せず補完関係(Trivyは「弱点の有無」、mdatpは「実際の攻撃・感染の有無」を監視)

### 監視方式

#### ① リアルタイム監視(常時)

- ファイルの作成・変更・実行等を常時監視し、侵入・感染を即時検知
- systemdで常駐プロセスとして稼働(`mdatp` サービス)

#### ② スケジュール型スキャン(日次)

- 毎日 **02:30** にフルスキャンを実行
- Trivy(02:00)・dump/転送/リストア処理と時間帯が重ならないよう調整
  - 02:00 Trivy 脆弱性スキャン
  - 02:30 mdatp フルスキャン
  - (dump/リストア処理は別時間帯で運用)

### M365 Defender Premiumとの統合

- 各VMをMicrosoft Defenderポータルにオンボーディングし、以下をクラウド側で一元管理
  - エージェントの稼働状況・シグネチャ更新状態
  - 検出インシデントの履歴・詳細調査
  - ポリシー(スキャン設定・除外設定等)の配布
- インシデント発生時はポータル上でのトリアージを一次窓口とし、Uptime Kuma/Slackは「気づき(一次アラート)」を担う位置づけ

### Uptime Kumaとの連携

- スケジュール型スキャン(日次02:30)の実行結果をPush通知
  - 成功時: `status=up`
  - マルウェア検出時: `status=down` + 検出内容をmsgに含めてSlackにも連携(Trivyと同様のパターン)
- エージェント自体の異常(プロセス停止、シグネチャ未更新等)もPush監視の対象とし、mdatpが停止した場合に気づけるようにする(常駐プロセスの死活)

### 実装ステップ

1. **インストール**
   - Microsoft公式リポジトリを追加
   - `apt-get install mdatp`
2. **オンボーディング**
   - Microsoft Defenderポータルで発行されるオンボーディングスクリプト(`MicrosoftDefenderATPOnboardingLinuxServer.py`等)を実行し、テナントに登録
3. **リアルタイム監視の有効化**
   ```bash
   mdatp config real-time-protection --value enabled
   ```
4. **スケジュールスキャンの設定**
   - cronで日次02:30にフルスキャンを実行
   ```bash
   # 毎日 02:30 に mdatp フルスキャンを実行
   30 2 * * * /opt/monitoring/mdatp/malware-scan.sh >> /var/log/mdatp-scan.log 2>&1
   ```
5. **Uptime Kuma統合**
   - スキャン完了後にPush URLへ結果を送信

### スクリプト配置と実行(案)

#### ディレクトリ構成

```
/opt/monitoring/mdatp/
├── malware-scan.sh        # スケジュールスキャン実行・結果通知スクリプト
└── logs/                  # スキャン結果ログ(簡易保管)
```

#### malware-scan.sh の概要

```bash
#!/bin/bash

# 1. フルスキャン実行(結果はmdatpのログ/ポータル側に集約される)
mdatp scan full

# 2. スキャン結果(脅威検出有無)を取得
THREATS=$(mdatp threat list --output json | jq '. | length')

# 3. 結果に応じてUptime Kumaへ通知
KUMA_URL="http://uptime-kuma:3001/api/push/mdatp-scan"
HOSTNAME=$(hostname)

if [ "$THREATS" -gt 0 ]; then
  STATUS="down"
  MSG="[mdatp Alert] $HOSTNAME: 脅威検出 $THREATS 件"
else
  STATUS="up"
  MSG="[mdatp OK] $HOSTNAME: 脅威検出なし"
fi

curl -X POST "$KUMA_URL?status=$STATUS&msg=$MSG&ping=$(date +%s)"

# 4. 脅威検出時はSlackにも通知
if [ "$STATUS" = "down" ]; then
  SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."
  curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"🦠 mdatpマルウェア検出アラート: $MSG\"}" \
    "$SLACK_WEBHOOK_URL"
fi
```

### 検出できる脅威 / できないもの

#### ✅ 検出できるもの

- 既知のマルウェア・ウイルス(シグネチャ/クラウドベース検知)
- 不審なファイル操作・実行(リアルタイム監視)
- 一部の未知の脅威(クラウド保護・ヒューリスティック検知)

#### ❌ 検出できないもの

- コンテナイメージ/OS/依存関係の既知脆弱性(CVE) → Trivyの担当領域
- アプリケーション側のロジック不備(SQLインジェクション等)
- 認可ロジックの不備、SCP/SSH認証情報の設定ミス

### 将来の拡張(保留事項)

- [ ] スキャン除外設定(パフォーマンスに影響しうるディレクトリの除外要否の検討)
- [ ] リアルタイム監視によるI/O負荷の実測・影響評価
- [ ] mdatpとTrivyのアラートをUptime Kuma上で集約ダッシュボード化
- [ ] インシデント対応Runbook(mdatp検出時の一次対応・エスカレーション先)の文書化

## 8. 未決定・今後の検討事項

- [ ] Runbook(障害検知 ~ DNS担当連絡 ~ 切替 ~ 復旧)の文書化
- [ ] 閾値超過時のリソースアラート実装(CPU/MEM/DISK)
- [ ] SCP/SSH用ユーザーのセキュリティ強化(command=制限、ラッパースクリプト等)
- [ ] DBネイティブなストリーミングレプリケーション/レプリカセット方式への移行(将来のRPO短縮)
- [ ] データ保全(誤操作対策・異常検知)のサニティチェック実装(dump前のデータ量基準値比較、閾値未決定)
- [ ] サニティチェックの閾値(現在10%想定)の運用調整
- [ ] Trivy 脆弱性DBの定期更新スケジュール確定
- [ ] 脆弱性スキップリスト(false positive対応)のレビュー・管理ポリシー決定
- [ ] CI/CD パイプラインへの Trivy 統合
- [ ] 旧カーネルパッケージの定期クリーンアップ手順の自動化検討
- [ ] 監視専用VM・スキャン対象VM双方のTrivyバージョン統一/更新管理手順の整備
- [ ] mdatp のスキャン除外設定・パフォーマンス影響評価
- [ ] mdatp と Trivy のアラートを統合したダッシュボード化
- [ ] mdatp 検出時のインシデント対応Runbook策定
- [ ] LVM/ZFSスナップショットによるファイルシステムレベルの巻き戻し機能検討
- [ ] 複数世代バックアップ保持(リソース許容時の検討)
- [ ] 物理的なオフサイト(別拠点)保管(リソース許容時の検討)
