# Mini AWS

AWSの仕組みを内部から学ぶための学習プロジェクト。EC2・ELB・SecurityGroupなどの主要機能をDockerコンテナで再現するWebアプリケーション。

## 技術スタック

| 領域 | 技術 |
|------|------|
| Backend | Go 1.24 / chi router |
| Frontend | React 18 / React Router v6 / Tailwind CSS |
| パッケージマネージャ | Bun 1.3 |
| ビルド | Vite |
| Lint (Go) | golangci-lint v2 |
| Lint (FE) | oxlint / knip |
| テスト (FE) | Vitest / Playwright |
| バリデーション | Zod |
| Git hooks | Lefthook |
| コンテナ | Docker API |
| バージョン管理 | mise |

## セットアップ

```bash
# mise でツールインストール
mise install

# バックエンド起動
go run cmd/server/main.go

# フロントエンド起動
cd frontend && bun install && bun run dev
```

## 学習の進め方

GitHub Issues に49個の課題が番号順に登録されています。Issue #1から順に実装してください。各Issueには以下の5項目が含まれます:

1. **実装内容** — 何を作るか
2. **技術説明** — AWSの内部的な仕組みの解説
3. **受け入れ条件** — 完了の判定基準
4. **ヒント** — 実装のとっかかり
5. **実装するテスト** — 書くべきテスト
