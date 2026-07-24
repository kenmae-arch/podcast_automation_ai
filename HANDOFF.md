# AIニュースポッドキャスト 開発引き継ぎ資料

このドキュメントは、**鹿島アントラーズ版ポッドキャスト(`~/Sandbox/App/podcast_automation`)と全く同じ要領で、AI業界ニュース版のポッドキャストを構築する**ためのClaude Code向け引き継ぎ資料です。鹿島版は2026-07-25に構築・配信開始済みで、その過程で得たノウハウとハマりどころをすべて記載しています。

## 1. プロジェクト概要

- **目的**: AI業界の最新ニュースを毎日ポッドキャスト配信する(完全無料運用)
- **方式**: 台本はLLM APIではなく**Claude Code自身が書く**(ユーザーの方針: API課金を避ける)
- **パイプライン**: Claude Codeがリサーチ→台本JSON作成 → `main.py` が Fish Audio でTTS → RSSフィード生成 → GitHub Pagesで配信 → Spotifyが自動取得
- **姉妹プロジェクト**:
  - `~/Sandbox/App/podcast_automation` — 鹿島版(このプロジェクトの雛形。**コードは全てここからコピーする**)
  - GitHub `kenmae-arch/daily-news-automation` — LINE配信版。AIニュースのリサーチ運用ルール(`news_ai.json` 側)はここのREADME準拠

## 2. 構築手順

### Step 1: コードのコピー

鹿島版からコード一式をコピーする(生成物・秘密情報・git履歴は除く):

```bash
cd ~/Sandbox/App/podcast_automation_ai
cp ~/Sandbox/App/podcast_automation/{main.py,config.py,utils.py,topic_fetcher.py,script_generator.py,audio_generator.py,rss_manager.py,requirements.txt,.gitignore,.env.example} .
mkdir -p docs/audio scripts .github/workflows
touch docs/audio/.gitkeep
cp ~/Sandbox/App/podcast_automation/.github/workflows/daily_podcast.yml .github/workflows/
```

コードは変更不要でそのまま動く(疎結合設計)。変更するのは設定ファイルとCLAUDE.md、辞書のみ。

### Step 2: .env の作成(★.env.exampleに実キーを書かないこと)

```bash
# Fish Audio APIキー(鹿島版と同じキーを使い回してOK。鹿島版の .env からコピー)
FISH_AUDIO_API_KEY=<鹿島版の.envから>

# ボイスID(ユーザーに確認。鹿島版と同じなら 5161d41404314212af1254556477c17d)
FISH_AUDIO_REFERENCE_ID=<ユーザーに確認>

LLM_PROVIDER=manual
SITE_BASE_URL=https://kenmae-arch.github.io/podcast_automation_ai

PODCAST_TITLE=AI INSIGHTS デイリー
PODCAST_DESCRIPTION=生成AI・機械学習・AI業界の最新ニュースと注目トピックを毎日お届けするAI生成ポッドキャストです。
PODCAST_AUTHOR=AI Insights Daily Podcast
PODCAST_EMAIL=k-maekawa-9jt@eagle.sophia.ac.jp
PODCAST_CATEGORY=Technology

# AI系ニュースの収集起点(Google News RSS検索。カンマ区切り)
NEWS_FEED_URLS=https://news.google.com/rss/search?q=%E7%94%9F%E6%88%90AI%20OR%20OpenAI%20OR%20Anthropic%20OR%20%E4%BA%BA%E5%B7%A5%E7%9F%A5%E8%83%BD&hl=ja&gl=JP&ceid=JP:ja
MAX_TOPICS=10
```

- 番組名・説明・カテゴリはユーザーに最終確認する(LINE版は「AI INSIGHTS デイリーニュース」、テーマカラーはディープネイビー `#1A2B4A`)
- ⚠️ 過去の事故: ユーザーがAPIキーを `.env.example` に書いてしまったことがある。キーは必ず `.env`(gitignore済み)へ。

### Step 3: CLAUDE.md の作成

鹿島版の `~/Sandbox/App/podcast_automation/CLAUDE.md` を雛形に、リサーチルールをAI業界向けに書き換える:

- **鮮度が最重要**: 原則直近24時間以内。各記事の公開日時を必ず確認。前日の内容は `scripts/published/` で確認し再掲禁止(新展開があるときのみ可)
- **情報源(AI版)**: Google News RSS検索(上記URL)、ITmedia AI+、日経クロステック、Publickey、ASCII.jp、各社公式ブログ(OpenAI/Anthropic/Google/Meta)、Hacker News上位。英語記事は和訳して採用可(参照元URLは原文)
- **広めのクエリでも検索**: 「生成AI 発表」「LLM 新モデル」「AI 資金調達」「AI 規制」等で当日の新規トピックを漏らさない
- **話題系も扱う**: X等で盛り上がっているAIツールの使い方、モデルの評判、面白い活用事例なども番組の柱
- **事実確認**: 記事本文・公式発表で裏付けてから台本化。憶測は「〜と報じられています」と出典明示
- **台本トーン**: テック好きのリスナーに話しかける「です・ます」調の1人語り、1200〜2000文字(3〜5分)、読み上げテキストのみ
- **読み仮名辞書の運用ルール**(そのままコピー): 読み間違い指摘があったら必ず `pronunciation_dict.json` に追記

### Step 4: pronunciation_dict.json(AI用語版)

TTS(Fish Audio)は英字・専門用語を読み間違えることがある。初期辞書の例:

```json
{
  "OpenAI": "オープンエーアイ",
  "ChatGPT": "チャットジーピーティー",
  "GPT": "ジーピーティー",
  "Anthropic": "アンソロピック",
  "Claude": "クロード",
  "Gemini": "ジェミナイ",
  "LLM": "エルエルエム",
  "AGI": "エージーアイ",
  "API": "エーピーアイ",
  "NVIDIA": "エヌビディア",
  "Hugging Face": "ハギングフェイス",
  "xAI": "エックスエーアイ",
  "Llama": "ラマ",
  "Meta": "メタ",
  "DeepMind": "ディープマインド",
  "GitHub": "ギットハブ",
  "推論": "すいろん"
}
```

- 英単語はFish Audioがそのまま英語読みできる場合もある。**初回生成後に必ず試聴して、読み間違いをユーザーに確認してもらう**こと(鹿島版では人名・地名の読み間違い指摘が入った)
- 適用処理は `audio_generator.py` に実装済み(長い語から優先置換)。コードコピーだけで動く

### Step 5: 動作確認(ローカル)

```bash
python3 -m venv venv && ./venv/bin/pip install -r requirements.txt
# ffmpegはローカルに無くてもOK(バイト連結フォールバック実装済み。Actions側ではffmpegインストール)
```

1. `topic_fetcher.py` でトピック取得できるか確認(feedparserのSSL問題はrequests経由で解決済み)
2. 台本を `scripts/pending.json` に書く(形式: `{"title", "description", "script"}`)
3. `./venv/bin/python main.py` を実行 → `docs/audio/episode_YYYY-MM-DD.mp3` と `docs/feed.xml` が生成される
4. MP3をユーザーに送って声・読み・内容の確認をもらう

### Step 6: GitHub リポジトリ化

```bash
git init -b main
# .claude/settings.local.json は .gitignore に追加してコミットから除外
git add -A && git commit
```

- ⚠️ **公開リポジトリの新規作成はClaude Codeの権限クラシファイアにブロックされる**(`gh repo create` もAPIも不可)。**ユーザーに `podcast_automation_ai` という名前でPublicリポジトリを作ってもらう**こと(https://github.com/new)。Publicなのは無料プランのGitHub PagesがPublic限定のため
- push時に `send-pack: unexpected disconnect` が出たら: `git config http.postBuffer 157286400` で解決(MP3が大きいため)
- push後の設定(これらはClaude Codeで実行可能):

```bash
# Pages有効化(main / docsフォルダ)
gh api repos/kenmae-arch/podcast_automation_ai/pages -X POST -f "source[branch]=main" -f "source[path]=/docs"
# シークレットと変数
gh secret set FISH_AUDIO_API_KEY --repo kenmae-arch/podcast_automation_ai --body "<キー>"
gh variable set FISH_AUDIO_REFERENCE_ID --repo ... --body "<ボイスID>"
gh variable set PODCAST_TITLE / PODCAST_DESCRIPTION / PODCAST_AUTHOR / PODCAST_EMAIL / PODCAST_CATEGORY も同様
```

- デプロイ後、`https://kenmae-arch.github.io/podcast_automation_ai/feed.xml` と音声URLが200を返すことを確認(反映まで1〜2分、404が続くのは正常)

### Step 7: カバーアート(★Spotify必須要件)

- フィードに `<itunes:image>` が無いとSpotifyの検証で弾かれる。`docs/cover.jpg`(1400〜3000px四方、RGB JPG)を用意する
- **ユーザーに画像を用意してもらうか確認**(鹿島版はユーザーがGemini生成画像を支給。AI版のテーマカラーはディープネイビー系が候補)。暫定ならPillowでテキストベースの仮カバーを生成してよい(鹿島版で実績あり、フォントは `/System/Library/Fonts/Hiragino Sans GB.ttc`)
- `rss_manager.py` に `fg.image()` + `fg.podcast.itunes_image()` 実装済み(コードコピーで対応済み)

### Step 8: Spotify登録(★最重要のハマりどころ)

Spotify for Creatorsは**外部RSSの直接登録が実質できない**UIになっている。鹿島版で確立した手順:

1. ユーザーが https://creators.spotify.com で番組を新規作成(Spotifyホスト型になる)
2. **エピソードを1本手動アップロード**(1本も無いと先に進めない)。タイトル・説明文はClaude Codeが用意して渡す
3. 番組公開後、**設定 → 「ポッドキャストのリダイレクト」** に `https://kenmae-arch.github.io/podcast_automation_ai/feed.xml` を入力して「リダイレクト」実行
4. これで番組URLはそのまま、ホストがGitHub Pagesに切り替わり、以降はpushだけで自動配信される

注意:
- リダイレクト欄で「URLが無効です」と出た場合、フィード側の不備(カバーアート欠落など)を疑う。ただし鹿島版では一時的な検証エラーの可能性もあった(最終的に同じURLで成功)。フィードを完全にしてから再試行させる
- 認証メールがフィードの `PODCAST_EMAIL` 宛に届く場合がある(だからユーザーの実メールにしておく)
- Spotify操作(ログイン必要)はすべてユーザーにやってもらう。Claude Codeは手順と文言を用意する

## 3. 日々の運用(構築完了後)

ユーザーが「今日のエピソードを作って」と言う → Claude Codeが:
1. リサーチ(CLAUDE.mdのルール準拠、WebSearch/WebFetchで裏付け)
2. `scripts/pending.json` に台本保存(新出の難読語は先に辞書へ)
3. `main.py` 実行 → MP3をユーザーに送付
4. `git add docs/ scripts/ && git commit && git push` → Spotifyに自動反映

## 4. ハマりどころ総まとめ(鹿島版の実績)

| 問題 | 解決策 |
|---|---|
| feedparserがSSLエラー(macOS) | requests経由で取得(実装済み) |
| Google Newsのリンク先が取得不可(JSリダイレクト) | 記事の裏付けはWebSearch/WebFetchで行う |
| TTSの固有名詞読み間違い | pronunciation_dict.json(実装済み)。指摘が来たら追記 |
| ffmpegローカルに無い | バイト連結フォールバック(実装済み) |
| `gh repo create`(public)がブロックされる | ユーザーにリポジトリ作成を依頼 |
| git push が disconnect | `git config http.postBuffer 157286400` |
| SpotifyにRSS入力欄が無い | ホスト型で作成→1本手動アップ→リダイレクト(上記Step 8) |
| Spotify「URLが無効です」 | カバーアート等フィード完全化→再試行 |
| APIキーの置き場所 | `.env` のみ。`.env.example` はプレースホルダー |

## 5. 参考: 鹿島版の実績値

- 台本1250〜1400文字 → 音声約3.5〜4分、MP3(128kbps mono)で3〜4MB
- TTSチャンク上限1500文字(1400字なら1チャンクで済む)
- Fish Audioモデルは `s2.1-pro-free` 固定(完全無料・フェアユース)。**変更禁止**
- 音声生成1回あたり60〜80秒
- GitHub Pages反映は push から1〜2分
