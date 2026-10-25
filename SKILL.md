# CVE脆弱性調査自動化スキル導入手順 (GitHub Copilot Agentモード専用)

本プロジェクトでは、GitHub Copilotの **Agentモード（自律型コマンド実行機能）** を利用し、チャット欄からワンコマンドで「最新CVE情報のネット取得」と「ソースコードの脆弱性検査」を連続して自動実行するスキルを導入します。

開発者はIntelliJのターミナルを触る必要も、外部ツールをメニューから呼び出す必要もありません。

---

## 🛠️ 仕組みの概要

1. **【開発者】** Copilot Chat（Agentモード）で `@workspace python scripts/fetch_cve.py CVE-202X-XXXX を実行して、生成された md ファイルを元に現在のコードを検査して` と指示する。
2. **【Copilot Agent】** 指示を受けて、ローカル環境の Python スクリプトを自律的に実行。
3. **【Pythonスクリプト】** ネットから最新のCVE情報を取得し、`.github/cves/current_target_cve.md` を生成・更新。
4. **【Copilot Agent】** スクリプトの実行完了（Success）を確認後、生成された md ファイルの内容を自動で読み込み、開いているソースコードのコンテキストと照らし合わせて脆弱性レビューを実行。

---

## 📦 1. 必要なファイルの配置

プロジェクトのルートディレクトリに、以下の2つのファイルを配置します。

### ① `scripts/fetch_cve.py`
Copilot Agentが呼び出すための軽量なCVE取得スクリプトです。

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
    else:
        print("CVE IDを指定してください。")
```

### ② `.github/copilot-instructions.md`
Agentモードがファイルを読み込んだ際の「検査基準」を定義する指示書です（これを置いておくことで、プロンプトを毎回長々と書く必要がなくなります）。

```markdown
# CVE脆弱性検査ガイドライン
- 修正・レビュー対象のCVE情報は、必ず `.github/cves/current_target_cve.md` から読み取ってください。
- 記載されている脆弱性の根本原因（CWEや危険なAPIの使い方、パース処理の不備など）を分析してください。
- 現在開いているソースコード（またはプロジェクト内の関連コード）に、同様の脆弱性を作り込んでいる箇所がないか精査してください。
- リスクが検知された場合は、具体的な箇所（ファイル名・行数）と、脆弱性を修正した「安全なコードの修正案」を出力してください。
```

---

## 🏃‍♂️ 2. 実際の日常的な使い方（スキル実行手順）

1. IntelliJで調査したいソースコードを開きます。
2. Copilot Chatを開き、トグルや設定から **Agentモード（または `@workspace` / agentic features）** が有効であることを確認します。
3. チャット欄に以下の **マジックプロンプト** を入力して送信します。

> **プロンプト：**
> `@workspace python scripts/fetch_cve.py CVE-2024-XXXX を実行して、更新された .github/cves/current_target_cve.md の内容を元に、このコードを検査して。`

4. **これだけで完了です。** Copilotが自動でPythonを裏で走らせ、「SUCCESS」の出力を確認したあと、流れるようにコードの脆弱性レビューと修正案の提示までを自動で行ってくれます。
