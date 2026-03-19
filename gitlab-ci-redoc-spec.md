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

## generate.sh との比較

|項目           |generate.sh（現行）  |GitLab CI（新方式）|
|-------------|-----------------|--------------|
|実行タイミング      |手動               |push自動        |
|ハッシュ比較       |あり               |なし            |
|リビジョン連番      |同日の連番            |pipeline連番    |
|`.hash_cache`|必要               |不要            |
|対象ブランチ制御     |`BRANCH_PATTERNS`|`rules` の正規表現 |