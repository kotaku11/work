# Spectral OpenAPI Lint - GitLab CI/CD 導入仕様

## 概要

Spectral を使用して OpenAPI 3.0 仕様書を GitLab CI/CD で自動 lint する構成。

-----

## ファイル構成

```
project/
├── .gitlab-ci.yml
├── .spectral.yml
└── openapi.yaml
```

-----

## 設定ファイル

### `.spectral.yml`

```yaml
extends:
  - spectral:oas
```

- `spectral:oas` は OpenAPI 2.0 / 3.0 / 3.1 を自動判別
- 仕様ファイルの `openapi:` フィールドを見てバージョンを判断するため、明示的な指定は不要

### `.gitlab-ci.yml`

```yaml
stages:
  - lint

spectral-lint:
  stage: lint
  image: stoplight/spectral:6
  script:
    - spectral lint openapi.yaml
        --ruleset .spectral.yml
        --format junit
        --output spectral-report.xml
    - spectral lint openapi.yaml
        --ruleset .spectral.yml
        --format pretty
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
  artifacts:
    when: always
    reports:
      junit: spectral-report.xml
    expire_in: 1 week
  allow_failure: false
```

-----

## 動作仕様

|項目      |内容                                         |
|--------|-------------------------------------------|
|使用イメージ  |`stoplight/spectral:6`                     |
|ルールセット  |`spectral:oas`（公式 OpenAPI ルール）             |
|実行タイミング |MR 作成・更新時 / デフォルトブランチへの push 時             |
|レポート形式  |JUnit（GitLab UI で確認可能）+ pretty（コンソール出力）    |
|lint 失敗時|パイプライン失敗としてマージをブロック（`allow_failure: false`）|
|レポート保持期間|1 週間                                       |

-----

## チェック内容（spectral:oas）

- `info` フィールドの必須項目（title, version）
- オペレーションへの `operationId` 付与
- レスポンスの定義漏れ
- 参照（`$ref`）の整合性
- タグの一貫性
- パラメータの重複

-----

## 今後の拡張

カスタムルールを追加する場合は `.spectral.yml` の `rules:` セクションに追記する。

```yaml
extends:
  - spectral:oas

rules:
  # カスタムルールをここに追加
```