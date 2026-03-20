=== ISSUE #1 ===
# [設定] mise設定（Go 1.24 + Bun 1.3）、lefthook設定、.gitignore、.editorconfig

## 実装内容
プロジェクト全体の開発環境基盤を構築する。バージョン管理ツール `mise` で Go 1.24 と Bun 1.3 を固定し、Git フック管理に `lefthook 2.1` を導入する。`.gitignore` でビルド成果物やOS固有ファイルを除外し、`.editorconfig` でチーム全体のコーディングスタイル（インデント、改行コード等）を統一する。

## 技術説明
**mise** は asdf 互換のポリグロットバージョンマネージャーで、`.mise.toml` に宣言的にツールバージョンを記述できる。プロジェクトディレクトリに入ると自動的に指定バージョンが有効になる。
- https://mise.jdx.dev/

**Lefthook** は高速な Git フックマネージャーで、`lefthook.yml` に pre-commit / pre-push 等のフックを宣言的に定義する。並列実行をサポートし、lint やテストをコミット前に自動実行できる。
- https://github.com/evilmartians/lefthook

**.editorconfig** はエディタ横断でコーディングスタイルを統一する仕組みで、ほぼすべての主要エディタがサポートしている。
- https://editorconfig.org/

## 受け入れ条件
- [ ] `.mise.toml` に Go 1.24 と Bun 1.3 が定義されている
- [ ] `mise install` でツールがインストールされる
- [ ] `lefthook.yml` に pre-commit フックが定義されている（最低限 lint を実行）
- [ ] `lefthook install` が成功する
- [ ] `.gitignore` に Go バイナリ、node_modules、.env、dist/ 等が含まれる
- [ ] `.editorconfig` に charset=utf-8、indent_style、end_of_line=lf が定義されている
- [ ] 新しいクローン後に `mise install && lefthook install` だけで環境が整う

## ヒント
```toml
# .mise.toml
[tools]
go = "1.24"
bun = "1.3"
golangci-lint = "2"
lefthook = "2.1"

[env]
GOBIN = "{{config_root}}/bin"
```

```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    go-lint:
      glob: "*.go"
      run: golangci-lint run ./...
    fe-lint:
      glob: "frontend/**/*.{ts,tsx}"
      root: "frontend/"
      run: bun run lint
```

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 2

[*.go]
indent_style = tab
```

## 実装するテスト
- `TestMiseInstall`: `mise install` が exit code 0 で完了することを確認するスクリプトテスト
- `TestLefthookInstall`: `lefthook install` が正常に完了することを確認
- 手動確認: `.editorconfig` 設定がエディタに反映されること


=== ISSUE #2 ===
# [BE] Goプロジェクト初期化、golangci-lint v2設定

## 実装内容
Go モジュールを初期化し、プロジェクトのディレクトリ構造を作成する。`golangci-lint v2` の設定ファイルを作成し、コード品質を自動的に担保する仕組みを整える。エントリーポイントとなる `cmd/server/main.go` の雛形を作成する。

## 技術説明
**Go Modules** は Go 公式の依存管理システムで、`go.mod` ファイルにモジュールパスと依存関係を宣言する。`go mod init` でプロジェクトを初期化する。
- https://go.dev/doc/modules/managing-dependencies

**golangci-lint v2** は Go 向けの統合 linter ランナーで、50以上の linter を一括管理できる。v2 では設定ファイル形式が刷新され、`linters` セクションで有効/無効を制御する。
- https://golangci-lint.run/

Go プロジェクトの標準的なディレクトリレイアウト:
- `cmd/` - メインアプリケーションのエントリーポイント
- `internal/` - 外部パッケージからインポートできないプライベートコード
- https://go.dev/doc/modules/layout

## 受け入れ条件
- [ ] `go.mod` が作成され、モジュール名が適切に設定されている
- [ ] `cmd/server/main.go` が存在し、`go build ./cmd/server` が成功する
- [ ] `internal/` 配下に `core/`, `ec2/`, `vpc/`, `elb/`, `api/`, `docker/` ディレクトリが存在する
- [ ] `.golangci.yml` が作成され、`golangci-lint run ./...` が成功する
- [ ] `gofmt` / `goimports` / `govet` / `errcheck` / `staticcheck` が有効化されている
- [ ] `go vet ./...` がエラーなしで通る

## ヒント
```bash
go mod init github.com/yourname/my_aws
mkdir -p cmd/server internal/{core,ec2,vpc,elb,api,docker}
```

```go
// cmd/server/main.go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	log.Printf("Starting Mini AWS server on :%s", port)
	if err := http.ListenAndServe(fmt.Sprintf(":%s", port), nil); err != nil {
		log.Fatal(err)
	}
}
```

```yaml
# .golangci.yml
version: "2"
linters:
  enable:
    - gofmt
    - goimports
    - govet
    - errcheck
    - staticcheck
    - unused
    - gosimple
    - ineffassign
  settings:
    goimports:
      local-prefixes: github.com/yourname/my_aws
```

## 実装するテスト
- `TestMainPackageBuild`: `go build ./cmd/server` が正常にビルドできることを確認
- `TestGolangciLint`: `golangci-lint run ./...` が exit code 0 で完了することを確認


=== ISSUE #3 ===
# [BE] 共通基盤: AWS風ID生成・エラーレスポンス・タグモジュール

## 実装内容
AWS API と同様の形式でリソースIDを生成する関数、統一的なエラーレスポンス構造体、タグ（Key-Value ペア）の管理モジュールを作成する。これらは EC2、VPC、ELB すべてのサービスで共通利用される基盤コードである。

## 技術説明
**AWS リソース ID** は `{prefix}-{hex}` 形式を取る。例: `i-0abcdef1234567890`（EC2インスタンス）、`sg-0abcdef1234567890`（セキュリティグループ）。プレフィックスはリソースタイプを示し、以降は16進数のランダム文字列（通常17桁）である。
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/resource-ids.html

**AWS エラーレスポンス** は統一的な XML/JSON 構造を持ち、`Code`（エラーコード）、`Message`（人間可読メッセージ）、`RequestId` を含む。本プロジェクトでは JSON 形式で返す。

**タグ** は AWS リソースに付与できるキーバリューペアで、最大50個まで設定可能。`Name` タグは AWS コンソールの表示名として特別扱いされる。
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html

## 受け入れ条件
- [ ] `GenerateID(prefix string)` 関数が `prefix-` + 17桁の16進数を返す
- [ ] 生成されるIDが十分にランダムで衝突しない（crypto/rand使用）
- [ ] `APIError` 構造体に `Code`, `Message`, `StatusCode` フィールドがある
- [ ] `NotFound`, `InvalidParameter`, `DuplicateResource` 等の定義済みエラーがある
- [ ] `Tag` 構造体と `Tags` 型（スライス）が定義されている
- [ ] タグから `Name` を取得するヘルパー関数が存在する
- [ ] 全関数にユニットテストがある

## ヒント
```go
// internal/core/id.go
package core

import (
	"crypto/rand"
	"fmt"
)

func GenerateID(prefix string) string {
	b := make([]byte, 9) // 9 bytes = 18 hex chars, we'll use 17
	if _, err := rand.Read(b); err != nil {
		panic(fmt.Sprintf("failed to generate random ID: %v", err))
	}
	return fmt.Sprintf("%s-%x", prefix, b)[:len(prefix)+1+17]
}
```

```go
// internal/core/errors.go
package core

import "net/http"

type APIError struct {
	Code       string `json:"code"`
	Message    string `json:"message"`
	StatusCode int    `json:"-"`
}

func (e *APIError) Error() string {
	return fmt.Sprintf("%s: %s", e.Code, e.Message)
}

var (
	ErrNotFound = func(resource, id string) *APIError {
		return &APIError{
			Code:       "InvalidParameterValue",
			Message:    fmt.Sprintf("The %s '%s' does not exist", resource, id),
			StatusCode: http.StatusNotFound,
		}
	}
)
```

```go
// internal/core/tags.go
package core

type Tag struct {
	Key   string `json:"Key"`
	Value string `json:"Value"`
}

type Tags []Tag

func (t Tags) GetName() string {
	for _, tag := range t {
		if tag.Key == "Name" {
			return tag.Value
		}
	}
	return ""
}
```

## 実装するテスト
- `TestGenerateID_Format`: 生成されたIDが `prefix-[0-9a-f]{17}` の形式であることを検証
- `TestGenerateID_Uniqueness`: 1000回生成して重複がないことを検証
- `TestGenerateID_Prefixes`: `i`, `sg`, `ami`, `key`, `elb` 等のプレフィックスで正しく動作することを検証
- `TestAPIError_Error`: Error()メソッドが正しいフォーマットの文字列を返すことを検証
- `TestTags_GetName`: Name タグがある場合とない場合の GetName() を検証
- `TestTags_GetName_Empty`: 空の Tags で GetName() が空文字を返すことを検証


=== ISSUE #4 ===
# [BE] Dockerクライアントラッパー作成と接続テスト

## 実装内容
Docker Engine API に接続するクライアントラッパーを作成する。コンテナの作成・起動・停止・削除・一覧取得・状態確認などの操作を抽象化したインターフェースを定義し、テスト時にモックと差し替え可能にする。

## 技術説明
**Docker Engine API** は Docker デーモンと通信するための RESTful API で、Unix ソケット（`/var/run/docker.sock`）または TCP 経由でアクセスできる。Go 公式クライアントライブラリ `github.com/docker/docker/client` を使用する。
- https://docs.docker.com/engine/api/

**Docker SDK for Go** は Docker Engine API の Go バインディングで、コンテナのライフサイクル管理、イメージ操作、ネットワーク管理等の機能を提供する。
- https://docs.docker.com/engine/api/sdk/

本プロジェクトでは EC2 インスタンスを Docker コンテナとして実装するため、Docker API はシステムの中核となる。インターフェースを定義してテスタビリティを確保することが重要。

## 受け入れ条件
- [ ] `DockerClient` インターフェースが定義されている
- [ ] `ContainerCreate`, `ContainerStart`, `ContainerStop`, `ContainerRemove`, `ContainerList`, `ContainerInspect`, `ContainerStats`, `ImageList`, `ImagePull` メソッドがインターフェースに含まれる
- [ ] 実装が Docker Engine API に正しく接続できる
- [ ] `Ping()` で Docker デーモンとの接続が確認できる
- [ ] コンテキスト（context.Context）を全メソッドで受け取る
- [ ] テスト用のモック構造体が定義されている

## ヒント
```go
// internal/docker/client.go
package docker

import (
	"context"
	"io"

	"github.com/docker/docker/api/types"
	"github.com/docker/docker/api/types/container"
	"github.com/docker/docker/api/types/image"
	"github.com/docker/docker/client"
)

type Client interface {
	Ping(ctx context.Context) error
	ContainerCreate(ctx context.Context, config *container.Config, hostConfig *container.HostConfig, name string) (container.CreateResponse, error)
	ContainerStart(ctx context.Context, containerID string) error
	ContainerStop(ctx context.Context, containerID string, timeout *int) error
	ContainerRemove(ctx context.Context, containerID string) error
	ContainerList(ctx context.Context, options container.ListOptions) ([]types.Container, error)
	ContainerInspect(ctx context.Context, containerID string) (types.ContainerJSON, error)
	ContainerStats(ctx context.Context, containerID string) (io.ReadCloser, error)
	ImageList(ctx context.Context, options image.ListOptions) ([]image.Summary, error)
	ContainerCommit(ctx context.Context, containerID string, options container.CommitOptions) (types.IDResponse, error)
	Close() error
}

type dockerClient struct {
	cli *client.Client
}

func NewClient() (Client, error) {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		return nil, err
	}
	return &dockerClient{cli: cli}, nil
}

func (d *dockerClient) Ping(ctx context.Context) error {
	_, err := d.cli.Ping(ctx)
	return err
}
```

## 実装するテスト
- `TestNewClient`: クライアントが正常に作成できることを検証
- `TestPing`: Docker デーモンへの接続が成功することを検証（Integration test、Docker環境がない場合はスキップ）
- `TestMockClient`: モッククライアントがインターフェースを満たすことを検証
- `TestContainerCreate_Mock`: モックを使ってコンテナ作成のロジックを検証


=== ISSUE #5 ===
# [BE] chi routerセットアップ、CORSミドルウェア、ヘルスチェックAPI

## 実装内容
HTTP ルーター `chi` を導入し、API のルーティング基盤を構築する。フロントエンド（別ポートで起動）からのリクエストを受けるための CORS ミドルウェアを設定する。サーバーの稼働状態を確認するヘルスチェックエンドポイント `GET /api/health` を実装する。

## 技術説明
**chi** は Go 標準の `net/http` と完全互換の軽量ルーターで、ミドルウェアチェーン、URLパラメータ、サブルーターなどをサポートする。`net/http` の `Handler` インターフェースをそのまま使えるため、標準ライブラリとの互換性が高い。
- https://github.com/go-chi/chi

**CORS (Cross-Origin Resource Sharing)** は、ブラウザが異なるオリジン間の HTTP リクエストを制御するセキュリティ機構。開発環境ではフロントエンド（例: localhost:5173）とバックエンド（例: localhost:8080）が異なるオリジンになるため、CORS ヘッダーの設定が必要。
- https://developer.mozilla.org/ja/docs/Web/HTTP/CORS

**ヘルスチェック API** は、ロードバランサーやモニタリングツールがサービスの稼働状態を確認するためのエンドポイント。通常は `200 OK` を返すシンプルなエンドポイントとして実装する。

## 受け入れ条件
- [ ] `chi.NewRouter()` でルーターが初期化されている
- [ ] CORS ミドルウェアで `localhost:5173` からのリクエストが許可されている
- [ ] `GET /api/health` が `{"status": "healthy"}` を返す
- [ ] レスポンスヘッダーに `Content-Type: application/json` が設定される
- [ ] リクエストログミドルウェアが設定されている（メソッド、パス、ステータスコード、所要時間）
- [ ] `cmd/server/main.go` から chi ルーターを使ってサーバーが起動する
- [ ] 不明なルートに対して適切な 404 レスポンスが返る

## ヒント
```go
// internal/api/middleware.go
package api

import (
	"net/http"

	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
	"github.com/go-chi/cors"
)

func NewRouter() *chi.Mux {
	r := chi.NewRouter()

	r.Use(middleware.Logger)
	r.Use(middleware.Recoverer)
	r.Use(middleware.RequestID)
	r.Use(cors.Handler(cors.Options{
		AllowedOrigins:   []string{"http://localhost:5173"},
		AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
		AllowedHeaders:   []string{"Accept", "Content-Type", "Authorization"},
		ExposedHeaders:   []string{"Link"},
		AllowCredentials: false,
		MaxAge:           300,
	}))

	r.Route("/api", func(r chi.Router) {
		r.Get("/health", healthCheck)
	})

	return r
}

func healthCheck(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"status":"healthy"}`))
}
```

## 実装するテスト
- `TestHealthCheck`: `/api/health` が 200 と正しい JSON を返すことを検証
- `TestHealthCheck_ContentType`: Content-Type ヘッダーが `application/json` であることを検証
- `TestCORS_AllowedOrigin`: `localhost:5173` からのリクエストに CORS ヘッダーが付与されることを検証
- `TestCORS_DisallowedOrigin`: 許可されていないオリジンからのリクエストが拒否されることを検証
- `TestNotFound`: 存在しないパスに 404 が返ることを検証


=== ISSUE #6 ===
# [FE] Bun + Vite + React + Tailwind初期化、oxlint/knip/vitest/playwright設定

## 実装内容
フロントエンドプロジェクトを `frontend/` ディレクトリに初期化する。Bun をパッケージマネージャーとして使用し、Vite + React 18 + TypeScript + Tailwind CSS の構成を作成する。コード品質ツール（oxlint, knip）、テストフレームワーク（vitest, playwright）を設定する。

## 技術説明
**Bun** は JavaScript/TypeScript のオールインワンツールキットで、パッケージマネージャー・バンドラー・テストランナーを含む。npm 互換でありながら大幅に高速。
- https://bun.sh/

**Vite** はフロントエンドビルドツールで、ESModules を活用した高速な開発サーバーと、Rollup ベースの本番ビルドを提供する。
- https://vitejs.dev/

**Tailwind CSS** はユーティリティファーストの CSS フレームワークで、HTMLに直接クラスを記述してスタイリングする。
- https://tailwindcss.com/

**oxlint** は Rust 製の高速 JavaScript/TypeScript linter で、ESLint の50-100倍高速に動作する。
- https://oxc-project.github.io/

**knip** は未使用のファイル・エクスポート・依存関係を検出するツール。
- https://knip.dev/

## 受け入れ条件
- [ ] `bun install` で依存関係がインストールされる
- [ ] `bun run dev` で Vite 開発サーバーが起動する（ポート5173）
- [ ] React 18 + TypeScript が動作し、ブラウザに「Mini AWS Console」と表示される
- [ ] Tailwind CSS が有効で、ユーティリティクラスが適用される
- [ ] `bun run lint` で oxlint が実行される
- [ ] `bun run knip` で未使用ファイルチェックが実行される
- [ ] `bun run test` で vitest が実行される
- [ ] `bun run test:e2e` で playwright が実行される
- [ ] `tsconfig.json` で `strict: true` が設定されている

## ヒント
```json
// frontend/package.json
{
  "name": "mini-aws-console",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "oxlint src/",
    "knip": "knip",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test"
  }
}
```

```ts
// frontend/vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
});
```

```json
// frontend/knip.json
{
  "entry": ["src/main.tsx"],
  "project": ["src/**/*.{ts,tsx}"],
  "ignore": ["src/vite-env.d.ts"]
}
```

## 実装するテスト
- `TestApp.test.tsx`: App コンポーネントがクラッシュせずにレンダリングされることを検証
- `TestViteBuild`: `bun run build` がエラーなく完了することを確認するスクリプトテスト
- `e2e/smoke.spec.ts`: ブラウザで `/` にアクセスして「Mini AWS」テキストが表示されることを検証


=== ISSUE #7 ===
# [FE] 共通Layoutコンポーネント（AWSコンソール風サイドバー・トップバー）

## 実装内容
AWS マネジメントコンソールを模した共通レイアウトコンポーネントを作成する。上部にサービス名とヘッダーバー、左側にナビゲーションサイドバー（EC2、VPC、ELB等のサービスリンク）、右側にメインコンテンツエリアを配置する。React Router v6 でページ遷移を設定する。

## 技術説明
**React Router v6** は React アプリケーションの宣言的ルーティングライブラリ。`<BrowserRouter>`, `<Routes>`, `<Route>` コンポーネントでURLとコンポーネントのマッピングを定義する。`<Outlet>` はネストされたルートの子コンポーネントをレンダリングする場所を示す。
- https://reactrouter.com/en/main

**AWS マネジメントコンソール** のUI構造:
- トップバー: サービス名、リージョン選択、アカウント情報
- サイドバー: サービス内のナビゲーション（Instances, Security Groups, AMIs 等）
- メインエリア: コンテンツ表示領域

**Tailwind CSS** のレイアウトユーティリティを使い、`flex`, `grid`, `h-screen`, `overflow-auto` 等でレスポンシブなレイアウトを構成する。

## 受け入れ条件
- [ ] トップバーに「Mini AWS Console」ロゴと擬似リージョン表示（ap-northeast-1）がある
- [ ] サイドバーに EC2（Instances, AMIs, Key Pairs）、VPC（Security Groups）、ELB（Load Balancers, Target Groups）のリンクがある
- [ ] サイドバーのリンクをクリックすると React Router で対応ページに遷移する
- [ ] 現在のページに対応するサイドバーリンクがハイライトされる
- [ ] レイアウトがレスポンシブで、サイドバーが固定幅、メインエリアが伸縮する
- [ ] `<Outlet>` でメインコンテンツが表示される

## ヒント
```tsx
// frontend/src/components/Layout.tsx
import { NavLink, Outlet } from 'react-router-dom';

const navItems = [
  {
    category: 'EC2',
    items: [
      { label: 'Instances', path: '/ec2/instances' },
      { label: 'AMIs', path: '/ec2/images' },
      { label: 'Key Pairs', path: '/ec2/key-pairs' },
    ],
  },
  {
    category: 'VPC',
    items: [
      { label: 'Security Groups', path: '/vpc/security-groups' },
    ],
  },
  {
    category: 'ELB',
    items: [
      { label: 'Load Balancers', path: '/elb/load-balancers' },
      { label: 'Target Groups', path: '/elb/target-groups' },
    ],
  },
];

export function Layout() {
  return (
    <div className="flex h-screen bg-gray-100">
      {/* Sidebar */}
      <nav className="w-60 bg-[#232f3e] text-white overflow-y-auto">
        <div className="p-4 text-lg font-bold border-b border-gray-600">
          Mini AWS
        </div>
        {navItems.map((group) => (
          <div key={group.category} className="mt-4">
            <div className="px-4 text-xs text-gray-400 uppercase tracking-wider">
              {group.category}
            </div>
            {group.items.map((item) => (
              <NavLink
                key={item.path}
                to={item.path}
                className={({ isActive }) =>
                  `block px-4 py-2 text-sm hover:bg-[#37475a] ${
                    isActive ? 'bg-[#37475a] border-l-4 border-[#ec7211]' : ''
                  }`
                }
              >
                {item.label}
              </NavLink>
            ))}
          </div>
        ))}
      </nav>

      {/* Main content */}
      <div className="flex-1 flex flex-col overflow-hidden">
        <header className="h-12 bg-[#232f3e] text-white flex items-center px-4 text-sm">
          <span className="ml-auto">ap-northeast-1</span>
        </header>
        <main className="flex-1 overflow-auto p-6">
          <Outlet />
        </main>
      </div>
    </div>
  );
}
```

## 実装するテスト
- `Layout.test.tsx`: Layout コンポーネントがサイドバーとメインエリアをレンダリングすることを検証
- `Navigation.test.tsx`: ナビゲーションリンクが正しい path を持つことを検証
- `ActiveLink.test.tsx`: 現在のルートに対応するリンクがアクティブスタイルを持つことを検証
- `e2e/navigation.spec.ts`: サイドバーのリンクをクリックしてページが遷移することを検証


=== ISSUE #8 ===
# [FE] APIクライアントモジュールとZodスキーマ定義

## 実装内容
バックエンドAPIとの通信を行う共通クライアントモジュールを作成する。fetch ラッパーでエラーハンドリング・レスポンスパースを統一し、Zod でレスポンススキーマを定義してランタイムバリデーションを行う。

## 技術説明
**Zod** は TypeScript ファーストのスキーマバリデーションライブラリ。スキーマ定義から TypeScript 型を自動推論できるため、API レスポンスの型安全性をランタイムでも保証できる。
- https://zod.dev/

API クライアントの設計パターン:
- **Base URL 設定**: 開発環境では Vite のプロキシ経由で `/api` にアクセス
- **エラーハンドリング**: HTTP ステータスコードに応じた統一的なエラー処理
- **レスポンスパース**: Zod スキーマでバリデーション + 型変換
- **リクエストヘルパー**: GET/POST/PUT/DELETE のショートカット関数

## 受け入れ条件
- [ ] `api/client.ts` に fetch ベースの API クライアントが実装されている
- [ ] `get<T>`, `post<T>`, `del<T>` 等のジェネリックなリクエスト関数がある
- [ ] API エラー時に統一的な `ApiError` 型で throw する
- [ ] `types/` ディレクトリに Zod スキーマが定義されている
- [ ] Instance, SecurityGroup, AMI, KeyPair, LoadBalancer 等のスキーマがある
- [ ] Zod スキーマから TypeScript 型が推論（`z.infer`）されている
- [ ] レスポンスの Zod バリデーション失敗時にわかりやすいエラーが出る

## ヒント
```ts
// frontend/src/api/client.ts
const BASE_URL = '/api';

export class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    message: string,
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

async function request<T>(
  method: string,
  path: string,
  body?: unknown,
): Promise<T> {
  const res = await fetch(`${BASE_URL}${path}`, {
    method,
    headers: body ? { 'Content-Type': 'application/json' } : {},
    body: body ? JSON.stringify(body) : undefined,
  });

  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new ApiError(res.status, err.code ?? 'Unknown', err.message ?? res.statusText);
  }

  return res.json() as Promise<T>;
}

export const api = {
  get: <T>(path: string) => request<T>('GET', path),
  post: <T>(path: string, body?: unknown) => request<T>('POST', path, body),
  put: <T>(path: string, body?: unknown) => request<T>('PUT', path, body),
  del: <T>(path: string) => request<T>('DELETE', path),
};
```

```ts
// frontend/src/types/instance.ts
import { z } from 'zod';

export const InstanceStateSchema = z.object({
  Code: z.number(),
  Name: z.enum(['pending', 'running', 'shutting-down', 'terminated', 'stopping', 'stopped']),
});

export const InstanceSchema = z.object({
  InstanceId: z.string(),
  InstanceType: z.string(),
  ImageId: z.string(),
  State: InstanceStateSchema,
  Tags: z.array(z.object({ Key: z.string(), Value: z.string() })),
  LaunchTime: z.string(),
  PrivateIpAddress: z.string().optional(),
  PublicIpAddress: z.string().optional(),
  SecurityGroups: z.array(z.object({
    GroupId: z.string(),
    GroupName: z.string(),
  })),
});

export type Instance = z.infer<typeof InstanceSchema>;
export type InstanceState = z.infer<typeof InstanceStateSchema>;
```

## 実装するテスト
- `client.test.ts`: 正常レスポンスが正しくパースされることを検証（fetch をモック）
- `client.test.ts#error`: エラーレスポンスで ApiError が throw されることを検証
- `schemas.test.ts`: 有効な Instance データが InstanceSchema でパースできることを検証
- `schemas.test.ts#invalid`: 不正なデータで ZodError が throw されることを検証


=== ISSUE #9 ===
# [FE] ダッシュボードトップページ（サービス概要表示）

## 実装内容
Mini AWS Console のトップページ（ダッシュボード）を作成する。各サービスのリソース数（インスタンス数、セキュリティグループ数、ロードバランサー数等）をカード形式で表示し、各サービスへのクイックリンクを提供する。

## 技術説明
**AWS マネジメントコンソール** のダッシュボードは、ユーザーのアカウント内のリソース状況を一覧表示する。リソース数、最近のアクティビティ、サービスヘルスなどが表示される。

本プロジェクトでは、複数の API エンドポイントを並行して呼び出し（`Promise.all`）、各サービスのリソース数を取得してカード形式で表示する。

**React の useEffect と useState** を使ったデータフェッチパターンを適用する。ローディング状態・エラー状態の管理も含む。

## 受け入れ条件
- [ ] `/` にアクセスするとダッシュボードが表示される
- [ ] EC2 Instances の数が表示される（running / stopped / total）
- [ ] Security Groups の数が表示される
- [ ] Load Balancers の数が表示される
- [ ] 各カードをクリックすると対応するサービスページに遷移する
- [ ] データ読み込み中はローディング表示がある
- [ ] API エラー時にエラーメッセージが表示される
- [ ] ヘルスチェック API のステータスが表示される

## ヒント
```tsx
// frontend/src/pages/Dashboard.tsx
import { useEffect, useState } from 'react';
import { Link } from 'react-router-dom';
import { api } from '../api/client';

interface ServiceSummary {
  instances: { total: number; running: number; stopped: number };
  securityGroups: number;
  loadBalancers: number;
  healthy: boolean;
}

export function Dashboard() {
  const [summary, setSummary] = useState<ServiceSummary | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchSummary() {
      try {
        const [instances, sgs, lbs, health] = await Promise.all([
          api.get<{ Instances: unknown[] }>('/instances').catch(() => ({ Instances: [] })),
          api.get<{ SecurityGroups: unknown[] }>('/security-groups').catch(() => ({ SecurityGroups: [] })),
          api.get<{ LoadBalancers: unknown[] }>('/load-balancers').catch(() => ({ LoadBalancers: [] })),
          api.get<{ status: string }>('/health').catch(() => ({ status: 'unhealthy' })),
        ]);
        setSummary({
          instances: {
            total: instances.Instances.length,
            running: instances.Instances.filter((i: any) => i.State?.Name === 'running').length,
            stopped: instances.Instances.filter((i: any) => i.State?.Name === 'stopped').length,
          },
          securityGroups: sgs.SecurityGroups.length,
          loadBalancers: lbs.LoadBalancers.length,
          healthy: health.status === 'healthy',
        });
      } catch (e) {
        setError('Failed to load dashboard data');
      } finally {
        setLoading(false);
      }
    }
    fetchSummary();
  }, []);

  if (loading) return <div className="text-center py-12">Loading...</div>;
  // ... render cards
}
```

## 実装するテスト
- `Dashboard.test.tsx`: ダッシュボードがリソースカードをレンダリングすることを検証
- `Dashboard.test.tsx#loading`: ローディング状態が表示されることを検証
- `Dashboard.test.tsx#error`: API エラー時にエラーメッセージが表示されることを検証
- `Dashboard.test.tsx#navigation`: カードクリックで正しいパスに遷移することを検証
- `e2e/dashboard.spec.ts`: ダッシュボードページが正しく表示されることを検証


=== ISSUE #10 ===
# [BE] Instanceデータモデルとステートマシン定義

## 実装内容
EC2 インスタンスのデータモデルを定義し、インスタンスの状態遷移を管理するステートマシンを実装する。AWS の実際の状態コードと状態名に準拠する。インメモリストア（`sync.RWMutex` + `map`）でインスタンスデータを管理する。

## 技術説明
**EC2 インスタンスの状態** は以下の通り:
| Code | Name          | 説明                               |
|------|---------------|------------------------------------|
| 0    | pending       | インスタンス起動準備中             |
| 16   | running       | 実行中                             |
| 32   | shutting-down  | 削除処理中                         |
| 48   | terminated    | 削除完了                           |
| 64   | stopping      | 停止処理中                         |
| 80   | stopped       | 停止済み                           |

- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html

**状態遷移ルール**:
- `pending` → `running`（起動完了）
- `running` → `stopping`（停止指示）→ `stopped`（停止完了）
- `running` → `shutting-down`（削除指示）→ `terminated`（削除完了）
- `stopped` → `pending`（再起動指示）→ `running`
- `stopped` → `shutting-down` → `terminated`

**sync.RWMutex** は Go の読み取り/書き込みロックで、複数の goroutine から安全にデータにアクセスするために使用する。読み取りは並行可能、書き込みは排他的。

## 受け入れ条件
- [ ] `Instance` 構造体に `InstanceId`, `InstanceType`, `ImageId`, `State`, `Tags`, `LaunchTime`, `SecurityGroups`, `KeyName`, `PrivateIpAddress` フィールドがある
- [ ] `InstanceState` 構造体に `Code` (int) と `Name` (string) がある
- [ ] 状態コード定数（`StatePending=0`, `StateRunning=16` 等）が定義されている
- [ ] 状態遷移の妥当性を検証する `ValidateTransition(from, to)` 関数がある
- [ ] 不正な状態遷移を拒否する（例: `terminated` → `running`）
- [ ] `InstanceStore` がスレッドセーフなインメモリストアとして実装されている
- [ ] `Save`, `FindByID`, `FindAll`, `Delete` メソッドがある

## ヒント
```go
// internal/ec2/instance.go
package ec2

import (
	"time"

	"github.com/yourname/my_aws/internal/core"
)

const (
	StatePending     = 0
	StateRunning     = 16
	StateShuttingDown = 32
	StateTerminated  = 48
	StateStopping    = 64
	StateStopped     = 80
)

var stateNames = map[int]string{
	StatePending:     "pending",
	StateRunning:     "running",
	StateShuttingDown: "shutting-down",
	StateTerminated:  "terminated",
	StateStopping:    "stopping",
	StateStopped:     "stopped",
}

type InstanceState struct {
	Code int    `json:"Code"`
	Name string `json:"Name"`
}

func NewState(code int) InstanceState {
	return InstanceState{Code: code, Name: stateNames[code]}
}

type Instance struct {
	InstanceId       string        `json:"InstanceId"`
	InstanceType     string        `json:"InstanceType"`
	ImageId          string        `json:"ImageId"`
	State            InstanceState `json:"State"`
	Tags             core.Tags     `json:"Tags"`
	LaunchTime       time.Time     `json:"LaunchTime"`
	PrivateIpAddress string        `json:"PrivateIpAddress,omitempty"`
	SecurityGroups   []GroupRef    `json:"SecurityGroups"`
	KeyName          string        `json:"KeyName,omitempty"`
	ContainerID      string        `json:"-"` // Docker container ID (internal)
}

type GroupRef struct {
	GroupId   string `json:"GroupId"`
	GroupName string `json:"GroupName"`
}

// validTransitions defines allowed state transitions
var validTransitions = map[int][]int{
	StatePending:     {StateRunning, StateShuttingDown},
	StateRunning:     {StateStopping, StateShuttingDown},
	StateStopping:    {StateStopped},
	StateStopped:     {StatePending, StateShuttingDown},
	StateShuttingDown: {StateTerminated},
}

func ValidateTransition(from, to int) bool {
	allowed, ok := validTransitions[from]
	if !ok {
		return false
	}
	for _, s := range allowed {
		if s == to {
			return true
		}
	}
	return false
}
```

```go
// internal/core/store.go
package core

import "sync"

type Store[T any] struct {
	mu    sync.RWMutex
	items map[string]T
}

func NewStore[T any]() *Store[T] {
	return &Store[T]{items: make(map[string]T)}
}
```

## 実装するテスト
- `TestNewState`: 各状態コードに対して正しい状態名が返ることを検証
- `TestValidateTransition_Valid`: 有効な遷移（running→stopping等）が true を返すことを検証
- `TestValidateTransition_Invalid`: 無効な遷移（terminated→running等）が false を返すことを検証
- `TestInstanceStore_Save`: インスタンスを保存して取得できることを検証
- `TestInstanceStore_FindByID_NotFound`: 存在しないIDで nil が返ることを検証
- `TestInstanceStore_FindAll`: 複数インスタンスが全件取得できることを検証
- `TestInstanceStore_Delete`: 削除後に取得できないことを検証
- `TestInstanceStore_Concurrent`: 並行アクセスでデータ競合が起きないことを検証（`go test -race`）
