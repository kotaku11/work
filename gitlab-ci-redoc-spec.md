# GitLab CI組み込み 仕様書

## 概要

|項目      |内容                              |
|--------|--------------------------------|
|トリガー    |対象ブランチへのpush（直接push・MRマージ後のpush）|
|MR時     |CI不動作                           |
|Runner種別|Docker                          |
|実行イメージ  |`alpine:latest`                 |
|転送方式    |SSH/SCP                         |
|SSHポート  |`30900`                         |

-----

## 処理フロー

```
対象ブランチへのpush（直接 or MRマージ）
  ↓
CI起動（Docker Runner）
  ↓
1. SSHクライアントをインストール
2. SSH秘密鍵をCI/CD変数から取得・設定
3. ブランチ名のラベル変換（/ → -）
4. リビジョン採番（YYYY-MM-DD-{CI_PIPELINE_IID}）
5. Nginxホストに転送先ディレクトリを作成（ポート30900）
6. openapi.yaml をSCPで転送（ポート30900）
  ↓
Nginxに即時反映
```

-----

## リビジョン採番仕様

|項目           |内容                                     |
|-------------|---------------------------------------|
|フォーマット       |`YYYY-MM-DD-NNN`                       |
|連番部分         |`CI_PIPELINE_IID`（プロジェクト内pipeline連番）で代替|
|ハッシュ比較       |なし（pushのたびに採番・配置）                      |
|`.hash_cache`|不要                                     |

-----

## ファイル配置先

```
/usr/share/nginx/html/specs/{ブランチラベル}/{YYYY-MM-DD-NNN}/openapi.yaml
```

例：

```
specs/main/2024-03-19-042/openapi.yaml
specs/release-v2/2024-03-15-038/openapi.yaml
```

-----

## .gitlab-ci.yml

```yaml
stages:
  - deploy

variables:
  NGINX_HOST: "nginx.example.com"
  NGINX_USER: "deploy"
  YAML_PATH: "docs/openapi.yaml"
  REMOTE_SPECS_DIR: "/usr/share/nginx/html/specs"

deploy-redoc:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -p 30900 $NGINX_HOST >> ~/.ssh/known_hosts
  script:
    - BRANCH=$(echo "$CI_COMMIT_REF_NAME" | sed 's/\//-/g')
    - DATE=$(date +%Y-%m-%d)
    - REV="${DATE}-$(printf '%03d' ${CI_PIPELINE_IID})"
    - DEST="${REMOTE_SPECS_DIR}/${BRANCH}/${REV}"
    - ssh -p 30900 ${NGINX_USER}@${NGINX_HOST} "mkdir -p ${DEST}"
    - scp -P 30900 ${YAML_PATH} ${NGINX_USER}@${NGINX_HOST}:${DEST}/openapi.yaml
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^main$|^release\/.*/'
      when: on_success
    - when: never
```

-----

## CI/CD変数（GitLab設定画面で登録）

|変数名              |内容                |Masked|
|-----------------|------------------|------|
|`SSH_PRIVATE_KEY`|Nginxホストへの秘密鍵     |✅     |
|`NGINX_HOST`     |NginxホストのIPまたはホスト名|      |
|`NGINX_USER`     |SSH接続ユーザー         |      |

※ `NGINX_HOST` / `NGINX_USER` は `.gitlab-ci.yml` の `variables` に直書きしても可。秘匿性が必要な場合はCI/CD変数に移動する。

-----

## 秘密鍵のセットアップ手順

### 1. 鍵ペアの生成

```bash
ssh-keygen -t ed25519 -C "gitlab-ci-redoc" -f ~/.ssh/gitlab_ci_redoc
# パスフレーズは空のままEnter
```

### 2. 公開鍵をNginxホストに登録

```bash
cat ~/.ssh/gitlab_ci_redoc.pub | ssh -p 30900 deploy@nginx.example.com \
  "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### 3. 秘密鍵をGitLab CI/CD変数に登録

```bash
cat ~/.ssh/gitlab_ci_redoc  # 内容をコピーしてGitLabに登録
```

GitLabの **Settings > CI/CD > Variables** で登録：

|項目       |値                                            |
|---------|---------------------------------------------|
|Key      |`SSH_PRIVATE_KEY`                            |
|Value    |秘密鍵の内容（`-----BEGIN...` から `-----END...` まで全て）|
|Masked   |✅ ON                                         |
|Protected|対象ブランチをProtectedにしている場合はON                   |

### 4. 疎通確認

```bash
ssh -i ~/.ssh/gitlab_ci_redoc -p 30900 deploy@nginx.example.com "echo connected"
```

-----

## Nginxホスト側セットアップ

```bash
# deployユーザーの作成
useradd -m deploy

# SSH公開鍵を登録
mkdir -p /home/deploy/.ssh
echo "<公開鍵>" >> /home/deploy/.ssh/authorized_keys
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys

# specs/ディレクトリの書き込み権限を付与
chown -R deploy:deploy /usr/share/nginx/html/specs
```

-----

## ブラウザUI仕様

### セレクトボックス検索機能

セレクトボックスに検索機能を追加するため、**choices.js** を導入する。

|項目    |内容                                       |
|------|-----------------------------------------|
|ライブラリ |choices.js                               |
|読み込み方式|CDN優先・失敗時はローカルファイルにフォールバック（ReDoc.jsと同じ方式）|
|検索対象  |ブランチ選択・リビジョン選択の両方                        |

#### select2 vs choices.js 選定理由

|項目     |select2           |choices.js |
|-------|------------------|-----------|
|依存     |jQuery必要          |**依存なし**   |
|CDN提供  |✅                 |✅          |
|オフライン対応|jQueryも別途ローカル配置が必要|**1ファイルのみ**|
|軽量さ    |やや重い              |**軽量**     |

現行構成がjQuery非使用のシンプルなHTMLのため、choices.jsを採用。

### ローカルファイルの配置（オフライン対応）

```bash
# JS
curl -o /usr/share/nginx/html/choices.min.js \
  https://cdn.jsdelivr.net/npm/choices.js/public/assets/scripts/choices.min.js

# CSS
curl -o /usr/share/nginx/html/choices.min.css \
  https://cdn.jsdelivr.net/npm/choices.js/public/assets/styles/choices.min.css
```

### index.html への組み込み

```html
<!-- choices.js CSS -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/choices.js/public/assets/styles/choices.min.css"
  onerror="this.href='/choices.min.css'">

<!-- choices.js JS -->
<script src="https://cdn.jsdelivr.net/npm/choices.js/public/assets/scripts/choices.min.js"
  onerror="this.src='/choices.min.js'"></script>
```

```javascript
// ブランチ選択
new Choices('#branch-select', {
  searchEnabled: true,
  searchPlaceholderValue: 'ブランチを検索...',
  itemSelectText: '',
});

// リビジョン選択
new Choices('#revision-select', {
  searchEnabled: true,
  searchPlaceholderValue: 'リビジョンを検索...',
  itemSelectText: '',
});
```

### UI表示イメージ

```
[ ブランチ ]
┌─────────────────────────┐
│ 🔍 ブランチを検索...     │
├─────────────────────────┤
│ main                    │
│ release-v2              │
│ release-v1              │
└─────────────────────────┘
```

-----

## generate.sh との比較

|項目           |generate.sh（現行）  |GitLab CI（新方式）|
|-------------|-----------------|--------------|
|実行タイミング      |手動               |push自動        |
|ハッシュ比較       |あり               |なし            |
|リビジョン連番      |同日の連番            |pipeline連番    |
|`.hash_cache`|必要               |不要            |
|対象ブランチ制御     |`BRANCH_PATTERNS`|`rules` の正規表現 |