---
title: びっくりドンキーで作った Android アプリ、Google Play「12 人 × 14 日」の壁を越えるためにテスターを募集します
tags:
  - Kotlin
  - Claude
  - AndroidDev
  - GooglePlay
  - クローズドテスト
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに

10 日ほど前にこんな記事を書きました：

[びっくりドンキーからAndroidアプリを作って、これからストア公開する話](https://qiita.com/ymatsuza/items/7d18a932aa171f49a352)

要約すると：

- GitJournal の 2 リポ目以降が **¥3,120** の有料化に納得できなくて
- びっくりドンキーで Claude Cowork（Dispatch）にプロンプト 1 発投げて
- ランチ中に Android アプリ「**GitMD**」を作った話

その後 10 日かけて **Google Play クローズドテスト** まで漕ぎ着けました。

ところがここで、最後にして最大の壁にぶつかります。**「12 人 × 14 日」の壁** です。

ついでに **テスターも募集しています**。詳細は本記事末尾に。

<img src="https://raw.githubusercontent.com/ymatsuza1128/qiita/main/images/playstore-icon-512.png" width="160">

---

## あの後やったこと（5/10 → 5/20 の 10 日間）

びっくりドンキーから帰ってから、ストア公開に向けて進めたこと：

| やったこと | 結果 |
|------|------|
| リリース署名・AAB ビルド | 14MB の `.aab` 生成 |
| Play Store 本人確認 | 完了 |
| プライバシーポリシー設置 | GitHub Pages で公開 |
| Firebase Crashlytics 導入 | クラッシュ追跡 ON |
| 広告 ID / IARC / データセーフティ申告 | 全部 OK |
| 内部テスト → クローズドテスト Alpha | versionCode 3 で配信 |
| **テスター 12 人集め** | **🔴 ここで詰まっている** |

最後の項目以外は、ほぼ全部 Claude Code が走ってくれました。自分は確認と Play Console の入力フォームをポチポチしただけです。

---

## 「12 人 × 14 日」の壁とは

2023 年 11 月以降、Google Play で **新規個人開発者** がアプリを本番公開するためには：

- クローズドテストで **12 人以上のテスター** を集める
- **連続 14 日間** インストール継続してもらう
- それを Google が確認したうえで「本番アクセス申請」を出す
- Google が OK を出したら、ようやく公開できる

要件: https://support.google.com/googleplay/android-developer/answer/14151465

**12 人。14 日。連続。**

「自作 Markdown エディタを 12 人に使わせる」だけなのに、これがびっくりするほど集まりません。

開発と申請が（AI のおかげで）爆速で進んだ分、最後に **「人を集める」** という一番アナログな工程だけが取り残されました。

---

## テスター募集 失敗集（投げた場所が全部死んだ話）

ストア審査が通っただけでは公開できないので、テスター 12 人を集めるべく、5/19 〜 5/20 にかけて手当たり次第に募集を投げました。

結果から書くと **全滅** です。

### 1. /r/AndroidDev Discord（`#promotion`）で削除された

「AndroidDev」を冠した有名 Discord に `#promotion` というチャンネルがあって、ここはアプリ宣伝 OK と思って投稿。

→ 翌日、モデレーターから DM が届きました：

> ルール 6（当サーバーのルールを違反する形で、Google Play テスターやクローズドテストの募集、取引、交換を行うこと）

`#promotion` という名前に騙されました。**「アプリの宣伝」は OK、「テスター募集」は禁止対象** でした。

**教訓**: AndroidDev コミュニティはテスター募集 NG が多い。Rules を読んでから投げる。

### 2. Reddit 3 投稿が AutoMod に消された

「Reddit の `r/AndroidAppTesters` と `r/alphaandbetausers` ならテスター募集 OK」という情報を頼りに、5/20 に 3 投稿。

| Subreddit | 投稿 ID | 結果 |
|---|---|---|
| r/AndroidAppTesters | `1thogmm` | ❌ AutoMod 削除 |
| r/alphaandbetausers | `1tho2ss` | ❌ AutoMod 削除 |
| r/alphaandbetausers | `1thohaw` | ❌ AutoMod 削除 |

**罠**: 投稿者本人には削除が見えません。自分のプロフィールから見ると「投稿成功」と表示されるんですが、**ログアウト状態（シークレットウィンドウ）で見ると「この投稿は Reddit のフィルターによって削除されました」** と書かれている。

原因（推定・自信度: 中）：

- 新規 Reddit アカウントで karma が圧倒的に足りない
- 同一テンプレを複数 sub に短時間連投 → クロスポスト判定
- そもそもテスター募集系の AutoMod 自体がスパム対策で厳しめに設定されている

**教訓**: 投稿後は **必ずシークレットウィンドウで生存確認**。本人視点だけ見ていると「投稿したのに反応ない」と勘違いし続けます。

### 3. 有料 Discord 申請が音沙汰なし

「Android クローズドテストコミュニティ」という月額 ¥1,000 の有料 Discord に申請フォームを送信。

→ 22 時間経過、応答なし（5/20 時点）。

決済前に対応速度がこれだと、入った後の質問対応にも不安が残ります。選択肢から外しました。

### 4. X（Twitter）はまだ生きているが、拡散しない

X はテスター募集 OK な数少ない場所。日英両言語で投稿し、`#ClosedTesting` `#14DayTest` も付けています。

ただ、フォロワーが少ないと拡散が弱い、という別の壁にぶつかります。**ここはまだ生存中、リプ・RT 歓迎です。**

---

## ここまでわかったこと

10 日間で得た教訓を雑に並べると：

- **「審査通れば公開できる」ではない**。本番アクセス申請のためにテスター 12 人 × 14 日が要る
- **AndroidDev 系コミュニティの `#promotion` を信じてはいけない**。テスター募集はテスター募集専用の場所で
- **Reddit の AutoMod は新規アカウントに容赦ない**。本人には削除が見えないので尚タチが悪い
- **「お金を払えば集まる」サービス**（Testers Community ¥1,999 / app 等）も存在するが、**多くは相互テスト前提**で、こちらも 12 人分のアプリを 14 日試す工数が発生する

つまり、開発を AI に任せて時間が浮いても、**最後の 14 日間は人力で誰かにインストールしてもらうしかない**、という構図でした。

---

## 【本題】GitMD クローズドテスター募集

ここから本題です。**GitMD のクローズドテスター、急募です。**

### GitMD とは

GitHub のリポジトリを Markdown ノートの保存先として使える Android アプリです。

| 機能 | 説明 |
|------|------|
| GitHub OAuth | Device Flow でログイン |
| 任意リポジトリの clone | 既存 GitHub リポをそのままノート保存先に |
| Markdown エディタ＋プレビュー | GitHub Flavored Markdown 対応 |
| Git 操作 | commit / push をアプリ内から |
| ノート一覧 4 ビュー | Standard / Journal / Card / Grid |
| 任意 Git ホスト＋ PAT | Codeberg 等 Gitea/Forgejo 系でも動作確認済み |

<img src="https://raw.githubusercontent.com/ymatsuza1128/qiita/main/images/gitmd-screenshot1.png" width="200"> <img src="https://raw.githubusercontent.com/ymatsuza1128/qiita/main/images/gitmd-screenshot2.png" width="200"> <img src="https://raw.githubusercontent.com/ymatsuza1128/qiita/main/images/gitmd-screenshot3.png" width="200"> <img src="https://raw.githubusercontent.com/ymatsuza1128/qiita/main/images/gitmd-screenshot4.png" width="200">

### 参加方法（2 ステップ）

**ステップ 1**: 以下の Google グループに参加リクエストを送る

```
https://groups.google.com/g/gitmd-testers
```

「Web 上の誰でも参加リクエスト可」設定にしているので、リクエストいただければ随時承認します。

**ステップ 2**: 承認後、以下のリンクから Play Store でクローズドテスト版をインストール

```
https://play.google.com/apps/testing/com.ymatsuza.gitmd
```

※ Play Store ログイン中の Google アカウントが、ステップ 1 で登録した Google アカウントと一致している必要があります。

### お願い

- インストール後、**14 日間はアンインストールしないでください**（Google の連続使用要件）
- 1 日 1 回でいいので、アプリを起動してみてください（テスト期間にカウントされます）
- バグ・要望は GitHub Issues / X DM / コメント欄、どこでも受け付けます
- 触ってみての率直な感想（「触りにくい」「ここで詰まった」等）が一番ありがたいです

### お礼

- 本番公開後、希望者には README にテスタークレジット記載
- アプリ本体は完全無料で公開予定（広告なし、課金なし）

---

## まとめ

びっくりドンキーで 2 時間で作ったアプリが、**「テスター 12 人 × 14 日」という最後の人力工程** で止まっています。

コード生成は AI に投げれば一瞬で済む時代になったのに、**最後に必要なのは生身の人間 12 人**、という、なんとも面白い構図でした。

> 着想から完成まで 2 時間。
> 本番公開までに必要な「人」は 12 人 × 14 日。

もしこの記事を読んで「ちょっと触ってみるか」と思った方がいれば、Google グループに参加リクエストを送ってもらえると、本当に助かります。

「GitHub をノートの保存先にしたい」「GitJournal の代替が欲しかった」という方には、刺さるはずです。

---

## 関連リンク

- 前回の記事: [びっくりドンキーからAndroidアプリを作って、これからストア公開する話](https://qiita.com/ymatsuza/items/7d18a932aa171f49a352)
- Google Group（テスター登録）: https://groups.google.com/g/gitmd-testers
- Play Store クローズドテスト: https://play.google.com/apps/testing/com.ymatsuza.gitmd
- Google Play クローズドテスト要件（公式）: https://support.google.com/googleplay/android-developer/answer/14151465
