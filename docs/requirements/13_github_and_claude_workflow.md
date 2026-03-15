## GitHub入門（このプロジェクト向け）＋ Claude / Claude Code 運用ガイド

このドキュメントは、GitHubに不慣れな方向けに「GitHubが何か」「どう使うか」を、**本プロジェクト（仕様書中心＋将来実装）** の運用に落とし込んで説明します。

---

## 0) まず前提：Git と GitHub は別物

- **Git**：ファイルの変更履歴を記録する仕組み（ローカルPCで動く）
  - 例：いつ・誰が・どのファイルを・どう変えたかを保存して、戻したり、分岐したりできる
- **GitHub**：Gitのリポジトリを **ネット上で共有・レビュー・管理** するサービス
  - 例：チームで共同編集、PRレビュー、Issue管理、リリース配布

このプロジェクトでは、まず **Gitで履歴管理**し、必要なら **GitHubで共有とレビュー**をします。

---

## 1) GitHubでよく使う用語（最小）

- **リポジトリ（repo）**：プロジェクトの入れ物（フォルダ一式＋履歴）
- **コミット（commit）**：変更のスナップショット（「ここまでの変更」を1単位で保存）
- **ブランチ（branch）**：変更の分岐（例：`main` とは別に作業用の枝を作る）
- **プルリクエスト（PR）**：変更を本流（main）に取り込む提案
  - 仕様書でもPRは便利（差分が見やすい／レビューできる）
- **Issue**：タスクや議論のスレッド（TBDの論点管理にも使える）
- **タグ/リリース**：まとまった節目（仕様書v0.1、プロトタイプv0.1など）

---

## 2) このプロジェクトでの「おすすめ運用」（仕様書＋実装の両対応）

### 2.1 ブランチ方針（シンプル）
- **`main`**：常に「読める」「矛盾が少ない」状態を維持（仕様書の本流）
- 作業は基本 **作業ブランチ** を作ってPRで取り込む
  - 例：`docs/field-effect`、`feat/prototype-ui` など

### 2.2 仕様書の変更フロー（例）
1. Issue（任意）を作る（例：「フィールド効果：解除仕様を確定する」）
2. ブランチを作る（例：`docs/field-effect-dispel`）
3. `docs/` を編集
4. `docs/09_open_questions.md`（TBD）と `docs/11_questions_and_proposals.md`（疑問→回答確定）に反映
5. PR作成 → 差分レビュー → `main` へマージ

### 2.3 実装の変更フロー（将来）
- **実装の根拠となる仕様書（どのmdか）** をPR本文に書く
- 仕様と実装がズレたら、原則 **仕様を先に更新**（もしくは `09_open_questions.md` にTBDとして明文化）

---

## 3) 引っ越し手順（このフォルダ → GitHub → Claude Code）

このセクションは「今あるローカルフォルダ（例：`c:\Cursor_CreateProjects\3vs3-CardBattle`）を、GitHubに載せて、Claude Codeで扱えるようにする」ための具体手順です。

### 3.0 事前チェック（最初の1回）
- **このフォルダに秘密情報が無いか**（APIキー、トークン、パスワード、個人情報、`.env` 等）
  - 見つかった場合は、コミット前に削除/マスクし、`.gitignore` に追加する
- **Gitをインストール済みか**
  - 未インストールなら「Git for Windows」をインストールする
  - インストール確認（PowerShell）：

```bash
git --version
```

### 3.1 GitHubで「空のリポジトリ」を作る（ブラウザ操作）
1. GitHubにログイン
2. 右上の「+」→ **New repository**
3. Repository name を入力（例：`3vs3-CardBattle`）
4. 公開範囲を選ぶ（Public / Private）
5. **重要**：次はチェックを外す（ローカルに既にある前提のため）
   - Add a README file（オフ）
   - Add .gitignore（オフ）
   - Choose a license（必要なら後で追加）
6. **Create repository**
7. 作成後に表示されるURLを控える
   - HTTPS：`https://github.com/<user>/<repo>.git`
   - SSH：`git@github.com:<user>/<repo>.git`

   3vs3 game
    - HTTPS：https://github.com/sabamiso-dev/CardGame-Dev.git
    - SSH：git@github.com:sabamiso-dev/CardGame-Dev.git

### 3.2 ローカルフォルダをGit管理にする（初回だけ）
PowerShellで、プロジェクト直下へ移動します。

```bash
cd "c:\Cursor_CreateProjects\3vs3-CardBattle"
```

次に、Gitを初期化します（初回だけ）。

```bash
git init
```

#### 3.2.1 Gitの「署名情報（ユーザー名/メール）」を設定（必要な場合）
初回コミット時に「ユーザー名/メールが無い」と言われたら設定します。

```bash
git config --global user.name "あなたの名前"
git config --global user.email "you@example.com"
```

（補足）GitHubアカウントのメールと同一である必要はありません。

### 3.3 最初のコミットを作る

```bash
git add .
git commit -m "Initial spec and Claude Code setup"
```

### 3.4 GitHubを「リモート」として登録してpushする
ブランチ名を `main` に揃えます。

```bash
git branch -M main
```

GitHubのURLを `origin` として登録します（URLは自分のものに置き換え）。

```bash
git remote add origin <GitHubのリポジトリURL>
```

初回push（アップロード）します。

```bash
git push -u origin main
```

#### 3.4.1 認証（ここで止まりやすい）
GitHubはパスワードpushを基本的に受け付けないため、以下のどちらかになります。

- **HTTPSの場合**：ブラウザが開いてログイン承認（推奨：Git Credential Managerが誘導）
- **SSHの場合**：SSH鍵の作成とGitHub登録が必要（慣れてからでOK）

うまくいかない場合は、エラーメッセージをそのまま貼ってください（対応手順を具体化します）。

### 3.5 GitHubに反映されたか確認
- ブラウザでGitHubのリポジトリを開き、`docs/` や `CLAUDE.md` が見えることを確認

---

## 4) 以後の使い方（ブランチ → PR → mainに取り込む）

### 4.1 作業ブランチを作って変更する（推奨）
`main` へ直接コミットしないで、作業ブランチで進めると安全です。

```bash
git checkout -b docs/some-change
# ファイル編集
git add .
git commit -m "docs: update spec"
git push -u origin docs/some-change
```

### 4.2 PR（プルリクエスト）を作る（ブラウザ操作）
1. GitHubのリポジトリ画面を開く
2. 「Compare & pull request」または「Pull requests」→「New pull request」
3. base：`main` / compare：`docs/some-change` を選ぶ
4. 変更内容を書く（仕様書なら「どのmdをどう変えたか」「TBDの更新」など）
5. Create pull request
6. 可能ならレビューしてから **Merge**（取り込み）

---

## 5) Claude / Claude Code での「管理方法」

### 5.1 Claude Code が読むプロジェクト指示（重要）

（このリポジトリは既に対応済み）
Claude Code には「プロジェクトの決まりごと」を置ける仕組みがあります。

- **プロジェクト共通メモリ**：`./CLAUDE.md`（このリポジトリに追加済み）
- **ルール分割**：`./.claude/rules/*.md`（このリポジトリに追加済み）
- **個人用（Gitに載せない）**：`./CLAUDE.local.md`（自動でgitignore推奨）

このプロジェクトでは以下を基本にしています。
- 未決事項（TBD）は `docs/09_open_questions.md` に集約
- 疑問→回答確定は `docs/11_questions_and_proposals.md` に集約
- 仕様更新時は関連ドキュメントの整合性も同時に取る

### 5.1.1 Claude Code へ「引っ越し」（開き方の手順）
1. Claude Code を起動
2. GitHubから取得する方法を選ぶ
   - すでにローカルにクローン済み：**フォルダを開く**（このリポジトリ直下）
   - まだローカルに無い：**GitHubからクローン**（UIの案内に従ってリポジトリを選択）
3. リポジトリ直下を開いた状態で作業する
   - `CLAUDE.md` と `.claude/rules/` があるため、プロジェクト指示が自動で効きます

### 5.2 GitHub上でのClaude運用（おすすめ）
- **PR単位でClaudeに依頼**する  
  - 例：「このPR差分で仕様矛盾がないかチェックして」「TBDが `09_open_questions.md` に集約できているか確認して」
- **Issue→PR** の流れで、仕様の議論ログを残す（後から追える）

### 5.3 Claude Code での作業のコツ
- Claude Codeに「何を更新するか」を最初に宣言し、**更新対象ファイルを明示**する
  - 例：`docs/06_turn_flow.md` と `docs/07_data_model.md` を整合させて更新、など
- 大きい変更は、まず `docs/11_questions_and_proposals.md` に疑問と回答案を入れてから、仕様本文へ反映すると手戻りが減る

---

## 6) 事故を防ぐチェックリスト（重要）

- **秘密情報をコミットしない**
  - APIキー、トークン、パスワード、個人情報、`.env` など
- **個人設定は `CLAUDE.local.md` に置く**（GitHubに上げない）
- **`main` へ直接コミットしない**（慣れるまではPR経由推奨）
- **仕様書の参照先のリンク切れ**を作らない（目次 `docs/README.md` を更新）

---

## 7) よくあるエラーと対処（最小）

### 7.1 `git` が見つからない
- 例：`git : 用語 'git' は...認識されません`
- 対処：Git for Windows をインストールし、PowerShellを再起動して `git --version` を確認

### 7.2 初回コミットでユーザー名/メールを求められる
- 対処：`3.2.1` の `git config --global user.name / user.email` を設定

### 7.3 pushで認証に失敗する（HTTPS）
- 対処：GitHubのパスワードではなく、ブラウザ認証（Git Credential Managerの誘導）を使う
- それでも失敗する場合：表示されたエラー文をそのまま貼る（環境によって手順が分かれる）

### 7.4 `remote origin already exists`
- 対処：既に登録済みなのでURL確認

```bash
git remote -v
```

URLを入れ替えるなら：

```bash
git remote set-url origin <GitHubのリポジトリURL>
```

