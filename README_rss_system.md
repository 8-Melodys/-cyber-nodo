# 📡 AIコピペ用RSSニュースシステム構築手順書

Android/iOS上のLinuxエミュレータ環境（UserLAnd Ubuntu）において、各種技術メディアのRSSフィードを取得し、AI（LLM）への入力に最適化されたプロンプト付きHTMLを自動生成してWebサーバー（Nginx）経由で配信するシステムの構築記録です。

APIキーの年齢制限という環境の制約をスマートに回避するため、**「Web画面上でワンタップコピー ➡️ 外部AI（無料版Gemini等）へペースト」**という手動データパイプライン（プランB）を採用しています。

---

## 1. システム構成（アーキテクチャ）
* **プラットフォーム:** OS環境 (UserLAnd Ubuntu)
* **Webサーバー:** Nginx
* **言語環境:** Python 3 (python3-venv による隔離仮想環境 `myenv`)
* **主要ライブラリ:** `feedparser` (RSS解析用)
* **データ構造:** ロジック（Pythonコード）とデータ（プロンプト・HTMLテンプレート）の分離構造

---

## 2. 環境構築・インストール手順

### Step 1: Webサーバーの導入と権限変更
Nginxをインストールし、Pythonスクリプトから公開ディレクトリへ直接HTMLを書き込めるようにパーミッションを変更します。
```bash
sudo apt update && sudo apt install nginx -y
sudo chmod 777 /var/www/html

```
### Step 2: Python仮想環境の構築とアクティベート
OS側のシステムパッケージ保護（PEP 668）を回避するため、独立した仮想環境を構築して入室します。
```bash
sudo apt install python3-venv -y
python3 -m venv myenv
source myenv/bin/activate

```
*(成功するとプロンプトの左端に (myenv) と表示されます)*
### Step 3: 依存ライブラリのインストール
```bash
pip install feedparser

```
## 3. ソースコードの配置・デプロイ
### 📄 ① プロンプト型ファイルの作成 (prompt.txt)
AIにインフラエンジニア目線で要約させるための「型」を定義します。
```bash
cat << 'EOF' > prompt.txt
# 役割
あなたは一流のシニアインフラエンジニア、および優秀な技術エディターです。
システム障害報告書（ポストモーテム）や技術ニュースの生データを読み込み、現場のインフラエンジニアが10秒で本質を理解できる「最高に実用的な要約」を作成してください。

# 入力データ
以下の技術文書を解析してください：
---
タイトル: {title}
本文/概要: {content}
---

# 出力フォーマット
必ず以下の構造と見出し（Markdown形式）で出力してください。無駄な挨拶や前置きは一切不要です。

## 💥 障害の概要
- [いつ、何が、どの規模で起きたかを1行で]

## 🔍 原因（根本原因）
- [なぜ起きたのか（設定ミス、バグ、アクセス集中など）技術的な要因を1〜2行で詳しく]

## 🛠️ どう対処・再発防止したか（冗長化・アーキテクチャの変更など）
- [どのような対応で復旧し、今後どう対策するかをインフラ目線で1〜2行で]

## 💡 この事例から学ぶ教訓（インフラエンジニアへのアセット）
- [他のシステムでも応用できる、設計や運用の注意点・チェックポイントを1行で]
EOF

```
### 🐍 ② 処理プログラムの作成 (fetch_news_html.py)
RSSを取得し、prompt.txtと合体させてNginxにデプロイするメインロジックです。
```bash
cat << 'EOF' > fetch_news_html.py
import feedparser

FEEDS = {
    "Cloudflare Blog": "[https://blog.cloudflare.com/rss/](https://blog.cloudflare.com/rss/)",
    "Hacker News (Top Stories)": "[https://news.ycombinator.com/rss](https://news.ycombinator.com/rss)"
}

HTML_TEMPLATE = """<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AIコピペ用プロンプト生成器</title>
    <style>
        body {{ font-family: sans-serif; background: #f0f2f5; padding: 20px; }}
        .container {{ max-width: 600px; margin: 0 auto; }}
        h1 {{ text-align: center; color: #333; }}
        .card {{ background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); margin-bottom: 20px; }}
        .media-name {{ font-weight: bold; color: #0066cc; margin-bottom: 10px; }}
        textarea {{ width: 100%; height: 300px; font-family: monospace; padding: 10px; box-sizing: border-box; border: 1px solid #ccc; border-radius: 4px; background: #fafafa; }}
        .btn {{ display: block; width: 100%; padding: 10px; background: #28a745; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; text-align: center; margin-top: 10px; }}
    </style>
</head>
<body>
    <div class="container">
        <h1>📡 AIコピペ用プロンプト生成器</h1>
        {cards}
    </div>
    <script>
        function copyText(id) {{
            var textarea = document.getElementById(id);
            textarea.select();
            document.execCommand('copy');
            alert('プロンプトをコピーしました！Geminiに貼り付けてね！');
        }}
    </script>
</body>
</html>
"""

CARD_TEMPLATE = """
        <div class="card">
            <div class="media-name">📦 {media_name} の最新記事</div>
            <textarea id="prompt-{idx}">{full_prompt}</textarea>
            <button class="btn" onclick="copyText('prompt-{idx}')">📋 このプロンプトをコピー</button>
        </div>
"""

def generate_html():
    try:
        with open("prompt.txt", "r", encoding="utf-8") as f:
            prompt_template = f.read()
    except FileNotFoundError:
        print("❌ prompt.txt が見つかりません。")
        return

    cards_html = ""
    idx = 0

    for media_name, url in FEEDS.items():
        feed = feedparser.parse(url)
        posts = feed.entries[:1]
        
        for entry in posts:
            title = entry.get("title", "タイトルなし")
            summary = entry.get("summary", entry.get("description", "本文なし"))
            full_prompt = prompt_template.format(title=title, content=summary)
            
            cards_html += CARD_TEMPLATE.format(
                media_name=media_name,
                idx=idx,
                full_prompt=full_prompt.strip()
            )
            idx += 1

    final_html = HTML_TEMPLATE.format(cards=cards_html)
    
    with open("/var/www/html/index.html", "w", encoding="utf-8") as f:
        f.write(final_html)
    
    print("✨ WebサーバーへのHTMLデプロイが完了しました！")

if __name__ == "__main__":
    generate_html()
EOF

```
## 4. 運用・実行手順
最新のニュースを取得してWebページを更新したいときは、以下のコマンドを実行します。
```bash
# 1. 仮想環境への入室
source myenv/bin/activate

# 2. スクリプトの実行（HTMLの自動更新）
python3 fetch_news_html.py

# 3. ブラウザで確認
# スマホのブラウザから http://localhost:8080 にアクセスする。

```
## 5. 今後の課題（将来のアップデート計画）
 1. **完全自動化（API連携）**: 年齢制限をクリア、または別ルートのAPIキー取得後、google-genai 拡張モジュールを組み込み、手動コピペの工程を完全撤廃する。
 2. **定時実行の自動化**: cron（Linuxのタイマー機能）を導入し、上記スクリプトを1日数回、自動で裏側で走らせる仕組みを構築する。
