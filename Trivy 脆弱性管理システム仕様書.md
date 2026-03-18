# Trivy 脆弱性管理システム 仕様書

## 1. 概要

社内 Ubuntu サーバ（10〜50台）を対象に、Trivy を用いた脆弱性スキャンを自動化し、結果を PostgreSQL に蓄積・Redash でダッシュボード可視化する仕組みを構築する。

-----

## 2. システム構成

```
対象サーバ群（10〜50台）
  └─ Trivy インストール済み
        ↓ Ansible でスキャン実行・JSON回収
管理サーバ（1台）
  ├─ Ansible
  ├─ PostgreSQL（trivy DB）
  └─ Python スクリプト（JSON → DB投入）
        ↓ リモート接続（5432ポート）
Redash サーバ（別サーバ・導入済み）
  └─ ダッシュボード
```

### サーバ役割一覧

|サーバ          |役割         |主要ソフトウェア                     |
|-------------|-----------|-----------------------------|
|対象サーバ（複数台）   |スキャン対象     |Trivy                        |
|管理サーバ（1台）    |スキャン制御・DB管理|Ansible / PostgreSQL / Python|
|Redashサーバ（1台）|可視化・ダッシュボード|Redash（導入済み）                 |

-----

## 3. ソフトウェア構成

|ソフトウェア            |バージョン|用途              |
|------------------|-----|----------------|
|Trivy             |最新版  |脆弱性スキャン         |
|Ansible           |最新版  |対象サーバへの一括実行・結果回収|
|PostgreSQL        |最新版  |スキャン結果の蓄積       |
|Python3 + psycopg2|最新版  |JSON→DB投入スクリプト  |
|Redash            |導入済み |ダッシュボード可視化      |

-----

## 4. インストール手順

### 4-1. Trivy（対象サーバ全台）

```bash
sudo apt install wget apt-transport-https gnupg -y

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | sudo apt-key add -

echo "deb https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt update && sudo apt install trivy -y
```

### 4-2. Ansible（管理サーバ）

```bash
sudo apt update
sudo apt install ansible -y
```

**インベントリファイル（`/etc/ansible/hosts`）**

```ini
[targets]
web01 ansible_host=192.168.1.101
web02 ansible_host=192.168.1.102
db01  ansible_host=192.168.1.103
# 対象サーバを追記
```

**SSH鍵認証設定**

```bash
# 管理サーバで鍵生成
ssh-keygen -t ed25519 -f ~/.ssh/trivy_key

# 対象サーバ全台に公開鍵を配布
ssh-copy-id -i ~/.ssh/trivy_key.pub user@192.168.1.101

# 接続確認
ansible all -m ping
```

### 4-3. PostgreSQL（管理サーバ）

```bash
sudo apt install postgresql postgresql-contrib -y
sudo systemctl enable --now postgresql
```

**DB・ユーザ作成**

```sql
CREATE DATABASE trivy;
CREATE USER trivy_user WITH PASSWORD 'your_password';
GRANT ALL PRIVILEGES ON DATABASE trivy TO trivy_user;
```

**テーブル作成**

```sql
CREATE TABLE trivy_results (
    id            SERIAL PRIMARY KEY,
    scanned_at    TIMESTAMP NOT NULL,
    hostname      TEXT NOT NULL,
    pkg_name      TEXT,
    pkg_version   TEXT,
    vuln_id       TEXT,
    severity      TEXT,
    title         TEXT,
    fixed_version TEXT
);
CREATE INDEX idx_scanned_at ON trivy_results(scanned_at);
CREATE INDEX idx_severity   ON trivy_results(severity);
CREATE INDEX idx_hostname   ON trivy_results(hostname);
```

**リモート接続許可（Redashサーバからの接続用）**

`/etc/postgresql/*/main/postgresql.conf`

```conf
listen_addresses = '*'
```

`/etc/postgresql/*/main/pg_hba.conf`

```conf
# Redashサーバの IPアドレスに合わせて変更
host    trivy    trivy_user    192.168.1.200/32    md5
```

```bash
sudo systemctl restart postgresql
```

-----

## 5. スキャン・DB投入の仕組み

### 5-1. DB投入スクリプト（`/usr/local/bin/trivy-to-db.py`）

```python
import json, glob, psycopg2
from datetime import datetime

REPORT_DIR = "/tmp/trivy_reports"
DB_CONFIG  = {
    "host": "localhost", "dbname": "trivy",
    "user": "trivy_user", "password": "your_password"
}

def main():
    conn = psycopg2.connect(**DB_CONFIG)
    cur  = conn.cursor()
    scanned_at = datetime.now()

    for path in glob.glob(f"{REPORT_DIR}/*.json"):
        hostname = path.split("/")[-1].replace(".json", "")
        with open(path) as f:
            data = json.load(f)

        rows = []
        for result in data.get("Results", []):
            for v in result.get("Vulnerabilities") or []:
                rows.append((
                    scanned_at, hostname,
                    v.get("PkgName"), v.get("InstalledVersion"),
                    v.get("VulnerabilityID"), v.get("Severity"),
                    v.get("Title"), v.get("FixedVersion"),
                ))

        cur.executemany("""
            INSERT INTO trivy_results
              (scanned_at, hostname, pkg_name, pkg_version,
               vuln_id, severity, title, fixed_version)
            VALUES (%s,%s,%s,%s,%s,%s,%s,%s)
        """, rows)
        print(f"[OK] {hostname}: {len(rows)}件登録")

    conn.commit()
    cur.close()
    conn.close()

if __name__ == "__main__":
    main()
```

### 5-2. Ansible Playbook（`/etc/ansible/trivy-scan.yml`）

```yaml
---
- name: Trivy スキャン実行・回収
  hosts: targets
  become: yes
  tasks:
    - name: スキャン実行
      command: >
        trivy fs --scanners vuln
        --severity CRITICAL,HIGH,MEDIUM
        --format json
        --output /tmp/trivy_result.json /
      ignore_errors: yes

    - name: 結果を管理サーバに回収
      fetch:
        src: /tmp/trivy_result.json
        dest: "/tmp/trivy_reports/{{ inventory_hostname }}.json"
        flat: yes

    - name: 一時ファイルを削除
      file:
        path: /tmp/trivy_result.json
        state: absent

- name: DBに投入
  hosts: localhost
  tasks:
    - name: DBに投入
      command: python3 /usr/local/bin/trivy-to-db.py
```

### 5-3. cron 設定（管理サーバ）

```bash
# /etc/cron.d/trivy-scan
# 毎週月曜 03:00 にスキャン実行
0 3 * * 1 root ansible-playbook /etc/ansible/trivy-scan.yml >> /var/log/trivy/ansible.log 2>&1
```

-----

## 6. Redash 設定

### 6-1. データソース登録

Redash画面の `Settings → Data Sources → New Data Source` で以下を設定する。

|項目      |値            |
|--------|-------------|
|Type    |PostgreSQL   |
|Name    |Trivy管理DB    |
|Host    |管理サーバのIPアドレス |
|Port    |5432         |
|Database|trivy        |
|User    |trivy_user   |
|Password|your_password|

### 6-2. クエリ一覧

**クエリ1: CRITICAL件数（カウンター）**

```sql
SELECT COUNT(*) AS critical_count
FROM trivy_results
WHERE severity = 'CRITICAL'
  AND scanned_at = (SELECT MAX(scanned_at) FROM trivy_results);
```

**クエリ2: HIGH件数（カウンター）**

```sql
SELECT COUNT(*) AS high_count
FROM trivy_results
WHERE severity = 'HIGH'
  AND scanned_at = (SELECT MAX(scanned_at) FROM trivy_results);
```

**クエリ3: 対象サーバ数（カウンター）**

```sql
SELECT COUNT(DISTINCT hostname) AS server_count
FROM trivy_results
WHERE scanned_at = (SELECT MAX(scanned_at) FROM trivy_results);
```

**クエリ4: サーバ別・重大度別件数（棒グラフ）**

```sql
SELECT
    hostname,
    severity,
    COUNT(*) AS count
FROM trivy_results
WHERE scanned_at = (SELECT MAX(scanned_at) FROM trivy_results)
  AND severity IN ('CRITICAL','HIGH','MEDIUM')
GROUP BY hostname, severity
ORDER BY hostname, severity;
```

**クエリ5: 週次推移（折れ線グラフ）**

```sql
SELECT
    DATE_TRUNC('week', scanned_at)::date AS week,
    severity,
    COUNT(*) AS count
FROM trivy_results
WHERE severity IN ('CRITICAL','HIGH')
GROUP BY week, severity
ORDER BY week;
```

**クエリ6: CRITICAL一覧テーブル（NVDリンク付き）**

```sql
SELECT
    scanned_at::date AS スキャン日,
    hostname         AS サーバ,
    '<a href="https://nvd.nist.gov/vuln/detail/' || vuln_id
      || '" target="_blank">' || vuln_id || '</a>' AS CVE,
    pkg_name         AS パッケージ,
    pkg_version      AS 現在Ver,
    fixed_version    AS 修正Ver,
    title            AS 概要
FROM trivy_results
WHERE severity = 'CRITICAL'
  AND scanned_at = (SELECT MAX(scanned_at) FROM trivy_results)
ORDER BY hostname;
```

### 6-3. ダッシュボードレイアウト

```
┌────────────┬────────────┬────────────┐
│ CRITICAL数 │   HIGH数   │ サーバ台数 │  ← カウンター × 3
├────────────┴────────────┴────────────┤
│         サーバ別件数（棒グラフ）      │
├─────────────────────────────────────┤
│         週次推移（折れ線グラフ）      │
├─────────────────────────────────────┤
│         CRITICAL一覧（テーブル）     │
└─────────────────────────────────────┘
```

### 6-4. クエリ自動更新スケジュール

|設定箇所         |設定値                   |
|-------------|----------------------|
|各クエリのSchedule|毎週月曜 04:00（スキャン完了1時間後）|
|ダッシュボードの自動更新 |Every 24 hours        |

-----

## 7. 運用フロー

```
毎週月曜 03:00
  └─ Ansible で対象サーバ全台スキャン（Trivy）
        ↓
  JSON を管理サーバに回収
        ↓
  Python スクリプトで PostgreSQL に INSERT
        ↓
毎週月曜 04:00
  └─ Redash クエリが自動更新
        ↓
  ダッシュボードで最新結果を確認
        ↓
  CRITICAL 検出 → 即時対応
  HIGH     検出 → 1〜2週間以内に対応
  MEDIUM   検出 → 1ヶ月以内に計画的に対応
```

-----

## 8. 対応優先度の目安

|Severity        |対応目安 |内容            |
|----------------|-----|--------------|
|CRITICAL        |即時〜3日|リモートから悪用可能など深刻|
|HIGH            |1〜2週間|影響範囲が広い       |
|MEDIUM          |1ヶ月以内|条件付き悪用        |
|LOW / NEGLIGIBLE|計画的に |実害ほぼなし        |

-----

## 9. 導入チェックリスト

```
□ 対象サーバ全台に Trivy インストール
□ 管理サーバに Ansible インストール・SSH鍵設定・hosts設定
□ 管理サーバに PostgreSQL インストール・DB/テーブル作成
□ PostgreSQL のリモート接続許可設定（pg_hba.conf / postgresql.conf）
□ DB投入スクリプト配置（/usr/local/bin/trivy-to-db.py）
□ Ansible Playbook 作成（/etc/ansible/trivy-scan.yml）
□ cron 登録（/etc/cron.d/trivy-scan）
□ Redash にデータソース登録（PostgreSQL接続確認）
□ Redash にクエリ作成（6本）
□ Redash ダッシュボード作成・レイアウト調整
□ クエリのスケジュール設定（毎週月曜 04:00）
□ 動作確認（手動でPlaybook実行 → DB確認 → Redash確認）
```