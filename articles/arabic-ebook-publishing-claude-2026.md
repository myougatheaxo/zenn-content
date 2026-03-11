---
title: "Claude Codeで作ったアラビア語電子書籍をKDPで売ってみた話"
emoji: "📚"
type: "idea"
topics: ["claudecode", "ai", "kdp", "電子書籍", "副業"]
published: true
---

## はじめに

「アラビア語で電子書籍を出版する」

一年前の自分にこれを言っても、笑われたと思う。アラビア語なんて一文字も読めないのに。

でも、Claude Codeと翻訳AI（ChatGPT）を組み合わせた結果、日本語で書いた原稿をアラビア語に翻訳して、KDP（Kindle Direct Publishing）にアップロードするところまでたどり着いた。

この記事では、その全プロセスを技術視点で解説する。

---

## なぜアラビア語か

世界で最もKindle電子書籍が少ない言語のひとつが、アラビア語だ。

アラビア語話者の人口は約4億人。GDP上位国にはサウジアラビア、UAE、カタールが入る。購買力がある市場なのに、コンテンツが圧倒的に少ない。

この非対称性が、チャンスだ。

日本語で書いた「自己啓発」「ビジネス」「お金の考え方」のような本は、アラビア語話者に刺さりやすいテーマでもある。

---

## 全体フロー

```
日本語原稿（Markdown）
    ↓
翻訳AI（ChatGPT / neonTetraAI）
    ↓
アラビア語原稿（manuscript_ar.md）
    ↓
EPUB ビルド（ebooklib + Amiri フォント）
    ↓
KDP アップロード
```

全てPythonで自動化している。

---

## 翻訳パイプラインの設計

### なぜChatGPTで翻訳するのか

アラビア語翻訳においては、Claude よりChatGPT（GPT-4o）の方が自然な文体になるケースが多かった。特にMSA（現代標準アラビア語）のフォーマル文体は、GPT-4oの方が安定している。

翻訳品質の比較検証をしてみた結果：

| 翻訳文 | 自然さ | アラブ文化適合性 | 処理速度 |
|--------|-------|----------------|---------|
| Claude Sonnet | B | B | ◎ |
| GPT-4o | A | A | ○ |
| 人間翻訳者 | A+ | A+ | △（遅い） |

月$20のChatGPT Plusで、20,000字の本が1時間で翻訳できる。人間に依頼すると数万円かかる作業だ。

### neonTetraAI の構成

ChatGPTのセッションを維持し、APIとして叩けるローカルサーバーを自作した（neonTetraAI: `localhost:9901`）。

```python
import requests

def translate_to_arabic(text: str) -> str:
    """日本語→アラビア語MSA翻訳"""
    resp = requests.post("http://localhost:9901/translate", json={
        "text": text,
        "prompt": "Translate to Arabic MSA. Output Arabic only, no explanations:"
    })
    return resp.json()["translated"]
```

翻訳ルール（メモリ化済み）：
- MSA（現代標準アラビア語）を使う
- 口語・方言は使わない
- アラブ文化的な格言を自然に補完する
- イスラムの文脈に適合するよう調整する

---

## EPUB ビルド

### Amiri フォントの重要性

アラビア語EPUBで最重要なのが、フォント埋め込みだ。

KDPやGoogle Play Booksは端末のフォントに依存するが、端末によってはアラビア語フォントが正しく表示されないケースがある。フォントを埋め込むことで、どの端末でも意図した表示になる。

**Amiri フォント**はArab Cultural Trust公認のオープンソースフォントで、商用利用可能かつ品質が高い。

```python
from pathlib import Path
from ebooklib import epub

FONTS_DIR = Path("products/books/fonts")
font_file = FONTS_DIR / "Amiri-Regular.ttf"

# フォントをEPUBに埋め込む
font_item = epub.EpubItem(
    uid="font-amiri",
    file_name="fonts/Amiri-Regular.ttf",
    media_type="application/x-font-ttf",
    content=font_file.read_bytes(),
)
book.add_item(font_item)
```

### RTL（右から左）対応

アラビア語はRTL（Right-to-Left）の言語だ。CSS設定が必要になる。

```css
body {
    direction: rtl;
    text-align: right;
    font-family: Amiri, serif;
}

h1, h2, h3 {
    direction: rtl;
}
```

HTML側でも `dir="rtl"` を忘れずに設定する。

```python
chapter_html = f"""
<html xmlns="http://www.w3.org/1999/xhtml"
      xml:lang="ar" lang="ar" dir="rtl">
<head>
  <meta charset="UTF-8"/>
  <link rel="stylesheet" href="../style/main.css"/>
</head>
<body>
{content}
</body>
</html>
"""
```

### 完成したEPUBの確認

```
marble-vol1-ar.epub: 251KB
├── fonts/Amiri-Regular.ttf  (453KB → EPUBに埋め込み)
├── style/main.css
├── chapter_00_front.xhtml
├── chapter_01_第1章.xhtml
├── ...
└── chapter_08_おわりに.xhtml
```

---

## カバー画像の生成

KDPのカバー画像は1600×2560ピクセル、JPEG形式が推奨。Pillowで自動生成している。

```python
from PIL import Image, ImageDraw, ImageFont
import io

def make_ar_cover(title: str) -> bytes:
    W, H = 1600, 2560
    img = Image.new("RGB", (W, H), color="#1a1a2e")
    draw = ImageDraw.Draw(img)

    # Amiriフォントでアラビア語タイトルを中央配置
    font = ImageFont.truetype("products/books/fonts/Amiri-Regular.ttf", 80)
    draw.text((W // 2, H // 2), title, font=font, fill="white", anchor="mm")

    buf = io.BytesIO()
    img.save(buf, format="JPEG", quality=90)
    return buf.getvalue()
```

---

## KDP登録の手順

1. **KDP管理画面** → 「新しいタイトルを追加」
2. 言語: Arabic を選択
3. EPUBファイルをアップロード（250KB程度でOK）
4. カバー画像をアップロード（1600×2560、JPEG）
5. 価格設定: $2.99〜$4.99 が売れやすいレンジ
6. 審査: 通常72時間以内に承認

Identity Verificationが必要になる場合がある（Amazonアカウントの本人確認）。税務情報の入力も事前に済ませておくこと。

---

## 実際の出版コスト・収益試算

| 項目 | コスト |
|------|-------|
| ChatGPT Plus | $20/月（翻訳に使い回し可） |
| KDP登録費 | 無料 |
| Amiriフォント | 無料（OSSライセンス） |
| 開発工数 | 初回20時間・2冊目以降3時間 |

KDPの印税率は70%（$2.99〜$9.99の場合）。

| 販売価格 | 手取り印税 | 月100冊売れた場合 |
|---------|-----------|---------------|
| $2.99 | $2.09 | $209 |
| $4.99 | $3.49 | $349 |

月100冊は控えめな目標だ。市場が薄い（競合が少ない）ため、SEOが効きやすい。

---

## 課題と解決策

### 課題1: アラビア語のレビューができない

自分ではアラビア語を読めないため、翻訳ミスを発見できない。

**解決策**: Google翻訳で逆翻訳（アラビア語→日本語）して原文と比較。完璧ではないが、致命的な誤訳は検出できる。

```python
def verify_translation(ja_text: str, ar_text: str) -> bool:
    """逆翻訳で品質確認"""
    back_translated = google_translate(ar_text, "ja")
    # キーワードが一致するか確認
    keywords = extract_keywords(ja_text)
    matches = sum(1 for k in keywords if k in back_translated)
    return matches / len(keywords) > 0.7
```

### 課題2: アラブ文化への配慮

自己啓発書は特に、文化的に不適切な表現が問題になる。

**解決策**: ChatGPTのメモリに「アラブ文化・イスラム配慮ルール」を保存し、全翻訳に適用させる。例えば：
- 「神に感謝」→ アラビア語版ではAlhamdulillahを自然に使う
- 家族・名誉（sharaf）の概念を尊重した表現
- ラマダン期間中はプロモーション内容を調整

---

## まとめ

アラビア語電子書籍出版のPythonスタックは以下の4つだ。

```
翻訳: neonTetraAI（ChatGPT API）
EPUB: ebooklib + Amiri フォント
カバー: Pillow
CI: Claude Code でスクリプト管理
```

「言語がわからない → AI翻訳 → eBOOK自動生成」というパイプラインは、2026年の現在では実用的なレベルに達している。

アラビア語市場は競合が少なく、技術的参入障壁を越えれば先行者メリットを享受できる。Claude Codeがあれば、この「技術的障壁を越える」部分は一人でもこなせる。

---

## 関連リソース

- [ebooklib ドキュメント](https://docs.sourcefabric.org/projects/ebooklib/)
- [Amiri フォント（alif-type）](https://github.com/alif-type/amiri)
- KDP（Kindle Direct Publishing）登録: kdp.amazon.com

---

*この記事で解説したパイプラインのコードは、自分のプロジェクト（wasabiAI）の一部として管理している。Claude Codeのカスタムスキルを使うと、このような多言語出版パイプラインの管理も自動化できる。*

*Security Pack（OWASP自動診断）・Code Review Pack（コードレビュー自動化）も PromptWorks で販売中。 → [prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。AIを使った多言語出版の実践記録。*
