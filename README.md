# 技術検証したいことリスト

## java開発環境の構築
* gradleマルチプロジェクト
* gretty
* spring
* junit
* jmockit

## sonarqube動作確認

## nessus動作確認

## e2eテスト動作確認
curl or seleniumによって、ログイン・参照・投稿・削除を実施する
* redmine
* growi
* nexus

## CI動作確認
* sonarqubeのCI化
* nessusのCI化
* e2eテストのCI化

## CD(継続的デリバリー)動作確認
git上のソースをサーバに転送する。

* dev環境
  * 不要
* staging/production環境
  * Push時に実施

## 自動デプロイ動作確認
`docker compose up -d`を実施して、dockerが起動完了し、e2eテストが通るところまで確認する。

* dev環境
  * 不要
* staging環境
  * 承認プロセスを必要としない
  * 切り戻しの確認
   * docker起動/e2eテストでエラーが発生した場合には、切り戻しする。
   * 一発で成功したとしても、最低一度は切り戻しができることを確認する
* production環境
  * 承認プロセスを必要とする
  * 何かしらエラーが発生した場合には、前のソースで再起動する

## 生成AIによるコードレビュー

---

# Todo List

## バックログ管理の検討
* Gitlab premium
* Planner
* Rychee Redmine

## [Retro]個人の環境での開発（動作確認）環境が整備されていないため、共有の環境を使っている。（リソースの無駄遣い）
* 案①:devlopment(個人)、staging(レプリカ)、production(マスタ)の運用を検討
* 案②:devlopment(個人)、staging(共有)、production(マスタ/レプリカ)の運用を検討

## [Retro]リリースが手動実施になっている。
* Gitlab-CIでPush時にデリバリー(資産転送)したい。
* Gitlab-CIでデプロイを自動化したい。

## 念のため、sonarqubeでのソース解析をやっておきたい。
* push時に、sonarqube解析を実施するようにしたい。

## 脆弱性検査を手動で行っている。
* Nessus使って、DockerコンテナおよびUbuntuの脆弱性検査を実施
 * Dockerの検査は、push時及び月例で実施したい。
 * Ubuntuの調査は、月齢で実施したい。

## Nexusのディスクサイズ肥大化
* 見直しを行う。

## 生成AIの活用
* コードレビュー
* UD
* UT
* CT
* PG
