# CVE脆弱性調査自動化スキル導入手順 (GitHub Copilot Agentモード最適化版)

本プロジェクトでは、GitHub Copilotの **Agentモード** と **Custom Instructions** を組み合わせることで、開発者が最小限の入力（例：「CVE-2024-XXXXに該当するか検査して」）を行うだけで、裏でスクリプト実行からコード解析までを自律的に連続実行するスキルを導入します。

---

## 📦 1. 必要なファイルの配置

プロジェクトのルートディレクトリに、以下の2つのファイルを配置します。

### ① `scripts/fetch_cve.py`
変更ありません。NVD APIから情報を取得し、特定のMarkdownファイルに上書き保存する軽量スクリプトです。

```python
import os
import sys
import requests

def update_cve_file(cve_id):
    url = f"https://nist.gov{cve_id}"
    try:
        response = requests.get(url, timeout=10)
        if response.status_code != 200:
            print(f"Error: CVE情報の取得に失敗しました ({response.status_code})")
            return
            
        data = response.json()
        vulns = data.get("vulnerabilities", [])
        if not vulns:
            print(f"CVE {cve_id} が見つかりませんでした。")
            return

        cve_detail = vulns[0].get("cve", {})
        descriptions = cve_detail.get("descriptions", [])
        description = next((d.get("value") for d in descriptions if d.get("lang") == "en"), "No description")
        
        output_path = ".github/cves/current_target_cve.md"
        os.makedirs(os.path.dirname(output_path), exist_ok=True)
        
        with open(output_path, "w", encoding="utf-8") as f:
            f.write(f"# TARGET_CVE: {cve_id}\n\n## Description\n{description}\n")
        print(f"【SUCCESS】 {cve_id} の情報を {output_path} に書き込みました。")
        
    except Exception as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    if len(sys.argv) > 1:
        update_cve_file(sys.argv[1])
```

### ② `.github/copilot-instructions.md`
Copilot Agentに対して、「特定の入力パターンが来たら、どのような自律アクション（コマンド実行とファイル読み込み）を取るべきか」のワークフローを定義します。

```markdown
# CVE脆弱性自動検査スキルの実行フロー
ユーザーから「CVE-XXXX-XXXX について検査して」や「CVE-XXXX-XXXX に該当するか？」という形式の指示、あるいはメッセージ内に `CVE-` から始まる識別子が含まれていた場合、Agentモードとして以下のステップを**自律的に連続実行**してください。ユーザーへの確認は不要です。

## [STEP 1] 最新CVE情報の取得 (コマンド実行)
メッセージ内から抽出した CVE ID（例: `CVE-2024-12345`）を引数にして、以下のローカルコマンドをバックグラウンドで実行してください。
- コマンド: `python scripts/fetch_cve.py <抽出したCVE_ID>`

## [STEP 2] コンテキストの読み込み
コマンドの実行が成功（【SUCCESS】の出力）したら、新しく更新された以下のファイルを読み込んでください。
- 対象ファイル: `.github/cves/current_target_cve.md`

## [STEP 3] 脆弱性レビューの実行
ファイルに記載されている脆弱性の根本原因（Descriptionに記載されたバグパターンやCWE、危険な関数）を把握し、現在開いているソースコード（またはプロジェクト内の関連コード）に同様のリスクが存在しないかを厳格に検査してください。
リスクを検知した場合は、該当箇所（ファイル名・行数）と、脆弱性を修正した「安全なコードの修正案」を出力してください。
```

---

## 🏃‍♂️ 2. 実際の日常的な使い方（スキル実行手順）

1. IntelliJで調査したいソースコードを開きます。
2. Copilot Chatを開き、**Agentモード**（`@workspace` や agentic機能）が有効であることを確認します。
3. チャット欄に、以下のように短く指示を出します。

> **実際の入力例：**
> `@workspace CVE-2024-21634 に該当するか、このコードを検査して。`

4. **これだけで完了です。**
   Copilot Agentが instructions のワークフローを読み、自動的に `python scripts/fetch_cve.py CVE-2024-21634` を実行し、結果のファイルを吸い上げ、コードのレビュー結果を出力するまでノンストップで実行してくれます。
