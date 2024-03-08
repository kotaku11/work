# Todo

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

## Nexusのディスクサイズが肥大化している。
* 見直しを行う。
