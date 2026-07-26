# Zenn Content

Zenn CLI と GitHub 連携を使って、記事を管理・公開するためのリポジトリです。

## 事前準備

- Node.js をインストールする
- Zenn のダッシュボードで、このリポジトリを GitHub 連携しておく
- リポジトリを clone し、依存パッケージをインストールする

```bash
npm install
```

## 記事を作成して投稿する

### 1. 記事ファイルを作成する

次のコマンドを実行すると、`articles` ディレクトリに Markdown ファイルが作成されます。

```bash
npx zenn new:article
```

ファイル名を指定したい場合は、12〜50 文字の半角英数字とハイフンを使った slug を指定します。

```bash
npx zenn new:article --slug my-first-article
```

### 2. 記事を執筆する

作成された `articles/<slug>.md` を編集します。ファイル先頭の Front Matter も設定してください。

```yaml
---
title: "記事のタイトル"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["zenn", "github"]
published: false
---
```

執筆中は `published: false` のままにします。`topics` は最大 5 個まで指定できます。

### 3. ローカルでプレビューする

```bash
npx zenn preview
```

コマンド実行後、ブラウザで <http://localhost:8000> を開き、表示を確認します。プレビューを終了するには、ターミナルで `Ctrl+C` を押します。

### 4. 記事を公開する

公開する記事の Front Matter を次のように変更します。

```yaml
published: true
```

変更をコミットし、Zenn と連携している GitHub リポジトリへ push します。

```bash
git add articles/<slug>.md
git commit -m "記事を追加: <記事タイトル>"
git push origin <ブランチ名>
```

Zenn CLI に記事を直接投稿するコマンドはありません。Zenn と GitHub の連携後、連携対象ブランチへ push された `published: true` の記事が Zenn に反映されます。反映状況やエラーは Zenn のダッシュボードで確認してください。

## 記事を修正する

### 公開前の記事を修正する

対象の `articles/<slug>.md` を編集し、`npx zenn preview` で表示を確認します。公開準備ができるまでは `published: false` を維持してください。

### 公開済みの記事を修正する

公開時と同じ `articles/<slug>.md` を編集します。`published: true` は変更せず、プレビューで確認してから変更を commit・push します。

```bash
npx zenn preview
git add articles/<slug>.md
git commit -m "記事を修正: <記事タイトル>"
git push origin <ブランチ名>
```

push 後、GitHub 連携によって Zenn 上の記事も更新されます。公開済み記事の slug（Markdown のファイル名）は記事 URL に使われるため、原則として変更しないでください。

### 記事を非公開にする

対象記事の Front Matter を `published: false` に変更して commit・push します。

```yaml
published: false
```

## 参考資料

- [Zenn CLI の使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [GitHub リポジトリで Zenn のコンテンツを管理する](https://zenn.dev/zenn/articles/connect-to-github)
