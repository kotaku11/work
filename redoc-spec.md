# ReDoc生成システム 最終仕様

## システム全体構成

```
[GitLab]                    [Dockerコンテナ]                 [ホストマシン/Nginx]            [ブラウザ]
ブランチ一覧取得  →  パターンマッチ → yaml取得(HTTP)  →  output/specs/ に配置  →  ReDoc.js でレンダリング
```

-----

## ファイル構成

```
redoc-gen/                          # 生成スクリプト一式
├── generate.sh                     # 実行エントリポイント
├── docker-compose.yml
├── .env                            # git管理外
├── .env.example
└── docker/
    ├── Dockerfile
    └── entrypoint.sh

/usr/share/nginx/html/              # Nginxドキュメントルート
├── index.html                      # ブラウザUI
├── redoc.standalone.js             # オフライン用ローカルJS
└── specs/                          # OUTPUT_DIRの出力先
    ├── main/
    │   ├── 2024-03-19-001/
    │   │   └── openapi.yaml
    │   └── 2024-03-18-001/
    │       └── openapi.yaml
    ├── release-v2/
    │   └── 2024-03-15-001/
    │       └── openapi.yaml
    └── release-v1/
        └── 2024-03-01-001/
            └── openapi.yaml
```

-----

## GitLab連携仕様

|項目      |内容                                                     |
|--------|-------------------------------------------------------|
|ホスティング  |セルフホスト（社内GitLab）                                       |
|プロトコル   |HTTP                                                   |
|ブランチ一覧取得|GitLab API (`/repository/branches?per_page=100&page=N`)|
|yaml取得  |GitLab API (`/repository/files/:path/raw?ref=:branch`) |
|認証      |アクセストークン（`PRIVATE-TOKEN` ヘッダー）                         |
|DNS解決   |コンテナ内 `/etc/hosts` にGitLabのIP/ホスト名を追記                  |

-----

## Dockerコンテナ仕様

|項目           |内容                                         |
|-------------|-------------------------------------------|
|ベースイメージ      |`alpine:3.19`                              |
|インストールパッケージ  |`curl` `ca-certificates` `jq`              |
|redoclyインストール|不要（レンダリングはブラウザ側のReDoc.jsが担う）               |
|処理内容         |DNS設定 → GitLab疎通確認 → yaml取得 → ホストマシンにマウント出力|

-----

## .env 設定項目

```bash
GITLAB_HOST=gitlab.example.com      # スキームなしのホスト名（HTTP接続）
GITLAB_TOKEN=glpat-xxxxxxxxxxxx     # アクセストークン（read_repository権限）
GITLAB_IP=192.168.1.100             # GitLabサーバーのIPアドレス
GITLAB_PROJECT_ID=123               # GitLabのプロジェクトID
YAML_PATH=docs/openapi.yaml         # リポジトリ内のyamlファイルパス

# 正規表現パターン（スペース区切りで複数指定）
# 形式: "正規表現パターン:ラベル生成ルール"
# __branch__ を指定するとブランチ名から自動生成（/ → - に変換）
BRANCH_PATTERNS="^main$:main ^release/.*:__branch__"

OUTPUT_DIR=./output                 # ホストマシンの出力先
```

### パターン指定例

```bash
# mainのみ
BRANCH_PATTERNS="^main$:main"

# release/ で始まる全ブランチを自動ラベル
BRANCH_PATTERNS="^release/.*:__branch__"

# main + release/* + feature/*
BRANCH_PATTERNS="^main$:main ^release/.*:__branch__ ^feature/.*:__branch__"

# v1, v2, v10 など v+数字 形式
BRANCH_PATTERNS="^v[0-9]+$:__branch__"
```

### **branch** のラベル変換例

```
release/v2   →  release-v2
release/v3   →  release-v3
feature/xyz  →  feature-xyz
```

-----

## リビジョン命名規則

```
YYYY-MM-DD-NNN

例: 2024-03-19-001       # その日の1回目
    2024-03-19-002       # 同日2回目（連番インクリメント）
```

-----

## Nginx設定仕様

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # autoindex は /specs/ のみに限定
    location /specs/ {
        autoindex on;
        autoindex_format json;       # Nginx 1.7.9以降が必要
        autoindex_exact_size off;
        autoindex_localtime on;
    }
}
```

-----

## ブラウザUI仕様

|機能           |内容                                            |
|-------------|----------------------------------------------|
|ブランチ選択       |セレクトボックス（Nginx autoindex JSONから動的取得）          |
|リビジョン選択      |セレクトボックス（ブランチ変更時に動的更新・降順ソート）                  |
|Latestバッジ    |各ブランチの最新リビジョン（先頭）に表示                          |
|URL直リンク      |`?branch=main&rev=2024-03-19-001` 形式          |
|ブラウザ履歴       |戻る/進むに対応（`pushState`）                         |
|ReDoc.js読み込み |CDN優先・失敗時はローカル `/redoc.standalone.js` にフォールバック|
|scrollYOffset|`52`（ツールバー高さ分オフセット）                           |

-----

## 運用フロー

```bash
# 初回セットアップ
cp .env.example .env
vi .env                             # 各変数を設定

# redoc.standalone.js をローカルに配置（オフライン対応）
curl -o /usr/share/nginx/html/redoc.standalone.js \
  https://cdn.jsdelivr.net/npm/redoc/bundles/redoc.standalone.js

# yaml更新時（手動実行）
./generate.sh
# → GitLab APIでブランチ一覧取得
# → BRANCH_PATTERNSにマッチしたブランチのyamlを取得
# → リビジョンを自動採番してNginxに即反映

# ブランチ追加時
# .env の BRANCH_PATTERNS が既存パターンにマッチすれば何もしない
# 新パターンが必要な場合のみ BRANCH_PATTERNS に追記して ./generate.sh を再実行
# → index.html・Nginx設定の変更は不要

# 差分確認時
# openapi-diffgenerator で任意の2リビジョンのyamlを直接比較（別途運用）

# リビジョン削除時
# 手動でディレクトリを削除（削除ポリシーなし）
rm -rf /usr/share/nginx/html/specs/main/2024-03-18-001
```

-----

## 将来対応予定

|項目       |内容                            |
|---------|------------------------------|
|CI/CD組み込み|現状手動実行。将来的にGitLab CI等への組み込みを予定|
|削除ポリシー   |現状手動管理。将来的に世代数・保持期間のルール化を検討   |