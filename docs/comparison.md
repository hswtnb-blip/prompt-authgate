# 関連研究・既存実装との比較

本実装（prompt-authgate）は新規性を主張しない。
以下の先行研究・既存実装と、それぞれの位置付けを整理する。

## 学術研究

### FATH: Formatting AuThentication with Hash-based tags

- 論文: [arXiv:2410.21492](https://arxiv.org/abs/2410.21492)（2024年10月）
- 手法: HMAC ベースの認証タグ。入力フォーマットに認証タグを埋め込み、LLM に出力の認証ラベル付けを指示し、ルールベースで検証
- 検証の主体: LLM 自身
- 対象: 間接的プロンプトインジェクションへの防御

### Signed-Prompt

- 論文: [arXiv:2401.07612](https://arxiv.org/abs/2401.07612)（2024年1月）
- 手法: 認可されたユーザーが命令セグメント内に署名。LLM が「信頼できる命令ソース」を識別する
- 検証の主体: LLM 自身
- 対象: LLM 統合アプリケーションへの直接的プロンプトインジェクション

### Defending Against Prompt Injection With a Few Defensive Tokens

- 論文: [ACM AISec Workshop 2025](https://dl.acm.org/doi/10.1145/3733799.3762982)
- 手法: 少数の防御トークンを挿入してモデルの堅牢性を強化

## OSS 実装

### marc-shade/claude-code-security

- リポジトリ: https://github.com/marc-shade/claude-code-security
- スコープ: Claude Code インストール全体の段階的ハードニング（Tier 1〜4）
- 機構:
  - AES-256-GCM 鍵保管庫
  - HMAC-SHA256 ファイル整合性署名
  - Ed25519 PKI、nonce リプレイ保護
  - Approval Tokens（HMAC-SHA256 + timestamp）
- 対象: ファイル／ノード／承認操作の認証

### fr0gger/nova-claude-code-protector (NOVA)

- リポジトリ: https://github.com/fr0gger/nova-claude-code-protector
- スコープ: セッション追跡と危険コマンドブロック
- 機構: 4 つの hook（SessionStart / PreToolUse / PostToolUse / SessionEnd）による 3 段階検出（キーワード / ML / LLM）
- 対象: 実行時のインジェクション検知（HMAC 認証は未使用）

### lasso-security/claude-hooks

- リポジトリ: https://github.com/lasso-security/claude-hooks
- スコープ: PostToolUse フックでツール出力をスキャン、疑わしい内容を警告

## prompt-authgate の位置付け

| 観点 | FATH / Signed-Prompt | marc-shade | NOVA | lasso | prompt-authgate |
|---|---|---|---|---|---|
| 検証主体 | LLM 内部 | 外部ハードニング層 | 実行時検知 | 実行時検知 | **LLM による判別（CLAUDE.md ルール）** |
| 認証対象 | プロンプト | ファイル／ノード | ツール呼び出し | ツール出力 | **ユーザー入力の送信元** |
| 機構 | 論文内実装 | 多層暗号システム | Python hook + ML/LLM | PostToolUse hook | **UserPromptSubmit hook 1 本** |
| 導入コスト | 研究用 | 本番対応（重厚） | 本番対応（要設定） | 軽量 | **超軽量（5 分）** |
| 対象ユーザー | 研究者 | エンタープライズ | 運用者 | 運用者 | **個人〜中小規模** |

## 相補関係

prompt-authgate は他の実装を置き換えるものではない。多層防御の一層として位置付けられる：

- **marc-shade** と **prompt-authgate** → 共存可（前者はファイル／ノード認証、後者は入力認証）
- **NOVA** と **prompt-authgate** → 共存可（前者は実行時検知、後者は入口認証）
- **FATH / Signed-Prompt** の思想 → prompt-authgate はその軽量実装版として機能

## 設計上の判断

### なぜ hook ベースか

Claude Code の `UserPromptSubmit` hook は、ユーザー入力時にだけ発火する。
この特性を利用すれば、外部プロキシや認証ゲートウェイを構築せずに
「入口の認証」を実装できる。

### なぜトークン 1 種類のみか

入口を **物理的に 1 つに絞る**（＝ hook 発火経路のみを信頼する）ことで、
鍵管理・ローテーション・検証ロジックを極小化できる。

複数レベルの認可が必要な運用では、
上位レイヤー（OAuth, SSO, marc-shade 等）と組み合わせる。

### 既知の限界

本実装は「モデルが CLAUDE.md のルールを守る」ことに依存する。
モデルの判断がルールを逸脱する可能性は原理的に残る。
したがって、**セキュリティ境界としてではなく、リスク低減策として**
位置付けることが正しい使い方である。
