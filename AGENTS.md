# AGENTS.md — AI INSIGHTS デイリー 運用マニュアル(エージェント共通)

このリポジトリは「AI INSIGHTS デイリー」というAI業界ニュースの日本語ポッドキャストを
毎日1本、自動生成・配信します。あなた(Codex 等のコーディングエージェント)の役割は
**リサーチ&台本担当**です。ニュースを集め、台本を書き、`scripts/pending.json` を
push するところまでが仕事です。**音声化と配信は GitHub Actions が自動で行います。**

> このファイルは「毎日の実行手順(ランブック)」を定めます。番組方針・リサーチルール・
> 台本トーン・読み仮名辞書の運用の**詳細は `CLAUDE.md`** にあります。着手前に
> **必ず `CLAUDE.md` も読んで従ってください。** トーン規定が矛盾する場合は `CLAUDE.md` を優先します。

## 日付の基準
- すべて**日本時間(JST / Asia/Tokyo)**で判断する。エピソードの日付 = 実行時点の JST の日付。

## 毎日のランブック
1. `CLAUDE.md` を読み、番組方針・リサーチルール・台本トーンを把握する。
2. `scripts/published/` の直近数日分の JSON を読み、前日までに扱った話題を確認する(**重複回避**)。
3. AI業界(生成AI・LLM・機械学習・主要AI企業・AI規制など)の**直近24時間**のニュース・話題を、
   利用可能な Web 検索/取得ツールで収集する。鮮度を最優先。新モデル発表・OpenAI / Anthropic /
   Google / Meta / xAI 等の発表・資金調達・規制・話題のツールを幅広く拾う。英語記事は和訳して
   採用可(参照元URLは原文のまま)。**各事実は記事本文や公式発表で裏付けてから使う。**
   憶測を事実のように書かない。噂・リークは出典と性質を明示する。
4. 日本語の台本を書く。トーンは `CLAUDE.md` 準拠(テック好きのリスナーに語りかける
   「です・ます」調の1人語り、専門用語はかみ砕く、目安1200〜2000字、読み上げ用テキストのみで
   見出し・記号・効果音指示は入れない、クロージングは「それではまた明日、AIの最前線でお会い
   しましょう」系)。**台本はエージェント自身が書く**(コスト回避のため、別の有料 LLM API で
   生成しない)。
5. TTS が読み間違えそうな**新出の固有名詞**(製品名・人名・企業名など)を
   `pronunciation_dict.json` に「表記→カタカナ読み」で追記する。台本JSON本体は元表記のままでよい
   (生成時に自動置換される)。ユーザーから読み間違いの指摘があれば必ず追記する。
6. `scripts/pending.json` に `{"title", "description", "script"}` の **UTF-8・整形 JSON** で保存する。
   `title` は日付を含む簡潔なもの、`description` は扱った話題を列挙した 200 字程度のショーノート。
7. `git add scripts/pending.json`(`pronunciation_dict.json` を変更したらそれも)して、
   **`Auto: daily episode <YYYY-MM-DD>`** のメッセージでコミットし、**`main` へ push** する。
   **push が完了条件**(push で GitHub Actions が起動し、音声化・配信まで自動で進む)。
8. **`main.py` は絶対に実行しない。** 音声生成・配信は GitHub Actions
   (`.github/workflows/daily_podcast.yml`)が担当する。CI 以外の環境には Fish Audio の API キーも
   venv も無く、ローカルでの音声生成は不要かつ不可能。Fish Audio モデルは `s2.1-pro-free` 固定(**変更禁止**)。
9. **直近24時間に新規の話題がまったく見つからない場合は、無理に古い話題を再掲せず、
   `pending.json` を作らずに終了**し、その日は配信スキップとして報告する。

## 配信の仕組み(触らないもの)
- `.github/workflows/daily_podcast.yml` は `scripts/pending.json` を含む push で起動 →
  `python main.py` を実行 → Fish Audio で音声化 → `docs/audio/` に保存、`docs/feed.xml` を更新、
  台本を `scripts/published/` へアーカイブする。
- したがってエージェントは `pending.json`(必要なら `pronunciation_dict.json`)を push するだけでよい。
  **`docs/` や `feed.xml`、`scripts/published/` は手で編集しない。**
- Fish Audio キー等の秘密情報は GitHub リポジトリの Secrets に設定済みで、CI のみが使用する。
  エージェント側で扱う必要はない。

## 環境メモ / 引き継ぎ上の注意
- **Web 取得の制約**: 実行環境によっては一部サイトへの直接 fetch や Google News RSS が
  ネットワークポリシーでブロックされることがある(Claude のクラウド環境では WebFetch と
  news.google.com が 403 だった)。その場合は**検索結果の見出し(検索インデックス由来)と
  複数クエリの相互確認**で裏取りし、各記事の**公開日時を必ず確認**する。Codex 環境では
  ポリシーが異なる可能性があるため、まず利用可能な取得手段を確認すること。
- **重複防止の要**: 手順 2 の `scripts/published/` 確認を毎回必ず行う。
- **推奨スケジュール**: これまで毎日 **00:00 JST 前後**に1回実行してきた(Claude 側 cron は
  `0 15 * * *` UTC = 00:00 JST)。Codex 側でも同程度の時刻に日次実行するとリズムが揃う。

## スケジュール実行用プロンプト(Codex に渡す想定・そのまま使える)
```
あなたは「AI INSIGHTS デイリー」ポッドキャストのリサーチ&台本担当です。
このリポジトリ直下の AGENTS.md と CLAUDE.md を読み、そこに定義された「毎日のランブック」に
従って、本日分(JST 基準)のエピソードを1本作成してください。手順の要点:
(1) scripts/published/ の直近数日を読み重複を避ける、
(2) 直近24時間の AI 業界ニュースを裏取りしながら収集する、
(3) CLAUDE.md 準拠のトーンで 1200〜2000 字の日本語台本を書く、
(4) 新出の難読固有名詞は pronunciation_dict.json に「表記→カタカナ読み」で追記、
(5) scripts/pending.json に {"title","description","script"} の整形 UTF-8 JSON で保存、
(6) `Auto: daily episode <YYYY-MM-DD>` でコミットし main へ push。
main.py は実行しないこと(音声化・配信は GitHub Actions が担当)。
直近24時間に新規の話題が皆無なら pending.json を作らずスキップし、その旨を報告すること。
最後に、タイトル・扱った話題・push の結果(またはスキップ理由)を簡潔に報告してください。
```
