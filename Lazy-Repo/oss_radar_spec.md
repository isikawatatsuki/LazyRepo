# oss_radar — Claude Code 実装プロンプト

## 指示

以下の仕様に従い `oss_radar` を実装せよ。  
不明点は実装を止めず、最もシンプルな解釈で進めること。

---

## 概要

GitHub上の活発なOSSリポジトリを発見するWebアプリ。  
GitHub OAuthでログインし、言語・トピック・活発さ指標でリポジトリを検索する。

---

## セットアップ・インストール

### 1. プロジェクト作成

```bash
# Vite + React + TypeScript テンプレートで初期化
bun create vite oss-radar --template react-ts
cd oss-radar
```

### 2. 依存パッケージのインストール

```bash
# ルーティング・状態管理
bun add react-router-dom zustand

# バックエンド（Hono）
bun add hono @hono/node-server

# Vercel アダプター
bun add @hono/vercel

# Tailwind CSS
bun add -d tailwindcss @tailwindcss/vite

# テスト
bun add -d vitest @vitest/ui jsdom @testing-library/react @testing-library/jest-dom

# Linter / Formatter
bun add -d @biomejs/biome
```

### 3. Tailwind CSS 初期化

`vite.config.ts` に Tailwind プラグインを追加:

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [react(), tailwindcss()],
})
```

`src/index.css` の先頭に追加:

```css
@import "tailwindcss";
```

### 4. Biome 初期化

```bash
bunx biome init
```

### 5. Vercel CLI インストール（デプロイ用）

```bash
bun add -D vercel
```

### package.json スクリプト例

```json
{
  "scripts": {
    "dev": "vercel dev",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest",
    "lint": "biome check .",
    "format": "biome format --write ."
  }
}
```

> **注意**: `bun run dev` ではなく `vercel dev` を使うことで、`api/` の Edge Functions をローカルでも動作確認できる。

---

## 技術スタック

| 項目 | 選定 |
|------|------|
| ランタイム | Bun |
| フロントエンド | Vite + React + TypeScript |
| スタイリング | Tailwind CSS |
| ルーティング | React Router v7 |
| 状態管理 | Zustand |
| バックエンド | Hono (Vercel Edge Functions) |
| 認証 | GitHub OAuth 2.0 |
| セッション管理 | httpOnly Cookie |
| Linter/Formatter | Biome |
| テスト | Vitest |
| デプロイ | Vercel |

---

## ディレクトリ構成

```
oss-radar/
├── src/
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   ├── OnboardingPage.tsx
│   │   ├── MainPage.tsx
│   │   └── SettingsPage.tsx
│   ├── components/
│   │   ├── RepoCard.tsx
│   │   ├── FilterSlider.tsx
│   │   └── PresetButtons.tsx
│   ├── hooks/
│   │   ├── useGitHubSearch.ts
│   │   └── useSettings.ts
│   ├── stores/
│   │   ├── authStore.ts      # ログインユーザー情報
│   │   └── settingsStore.ts  # フィルター設定
│   ├── lib/
│   │   └── github.ts         # API クライアント (fetch wrapper)
│   ├── types/
│   │   └── github.ts
│   ├── App.tsx
│   └── main.tsx
├── api/
│   └── [[...route]].ts       # Hono エントリーポイント
├── biome.json
├── vercel.json
├── vite.config.ts
├── .env.example
└── package.json
```

---

## 環境変数

```env
# .env.example
GITHUB_CLIENT_ID=your_client_id
GITHUB_CLIENT_SECRET=your_client_secret
SESSION_SECRET=random_32byte_string        # Cookie署名用
APP_URL=https://your-app.vercel.app
```

GitHub OAuth App の登録:
- URL: https://github.com/settings/developers
- Authorization callback URL: `{APP_URL}/api/auth/callback`
- スコープ: `public_repo` のみ

---

## 画面フロー

```
LoginPage
  └─ 「GitHubでログイン」ボタン
       └─ GitHub OAuth → /api/auth/callback
            └─ 初回: OnboardingPage（searchMode 選択）
            └─ 再訪: MainPage
                  └─ SettingsPage（ヘッダーの設定アイコンから遷移）
```

---

## 認証フロー

1. `LoginPage` で `https://github.com/login/oauth/authorize?client_id={CLIENT_ID}&scope=public_repo` へリダイレクト
2. GitHub から `GET /api/auth/callback?code=XXX` にリダイレクトされる
3. Hono Function でコードをアクセストークンに交換
4. トークンを httpOnly Cookie に保存（`secure: true, sameSite: 'lax', maxAge: 7days`）
5. 以降の GitHub API リクエストはすべて `/api/github/search` 経由（サーバー側でトークン付与）
6. `GET /api/auth/me` でフロントがログイン状態を確認（Cookie → GitHub user API）

---

## Hono ルート定義

```typescript
// api/[[...route]].ts
import { Hono } from 'hono'
import { handle } from 'hono/vercel'
import { setCookie, getCookie, deleteCookie } from 'hono/cookie'

export const config = { runtime: 'edge' }

const app = new Hono().basePath('/api')

app.get('/auth/callback', ...)  // code → token → Cookie → redirect
app.post('/auth/logout', ...)   // Cookie 削除
app.get('/auth/me', ...)        // Cookie → GitHub /user
app.get('/github/search', ...)  // GitHub Search API プロキシ

export default handle(app)
```

---

## API 仕様

### GET /api/auth/me

Cookie のトークンで GitHub `/user` を叩き、ユーザー情報を返す。  
Cookie なし or トークン無効 → `401`

**レスポンス**
```typescript
{ login: string; avatar_url: string }
```

### GET /api/auth/callback

| 処理 | 内容 |
|------|------|
| code 受取 | `?code=XXX` |
| token 交換 | `POST https://github.com/login/oauth/access_token` |
| Cookie 設定 | `httpOnly: true, secure: true, sameSite: 'lax', maxAge: 604800` |
| リダイレクト | `APP_URL + /` |

### POST /api/auth/logout

Cookie を削除して `200` を返す。

### GET /api/github/search

GitHub Search API のプロキシ。Cookie のトークンをサーバー側で使用。

**クエリパラメータ**

| パラメータ | 型 | 必須 | 説明 |
|------------|----|------|------|
| mode | `language \| topic \| both` | ✓ | 検索モード |
| language | string | | プログラミング言語 |
| topic | string | | GitHub トピック |
| minStars | number | | Star数下限 |
| minForks | number | | Fork数下限 |
| commitWithin | number | | 最終コミット（日数、0=制限なし） |
| page | number | | ページ番号（デフォルト: 1） |

**GitHub Search API クエリ構築**

```
is:public archived:false
[language:{language}]      ← mode が language or both かつ language 指定時
[topic:{topic}]            ← mode が topic or both かつ topic 指定時
[stars:>={minStars}]       ← minStars > 0 の場合
[forks:>={minForks}]       ← minForks > 0 の場合
[pushed:>{YYYY-MM-DD}]     ← commitWithin > 0 の場合
sort=stars&order=desc&per_page=20&page={page}
```

**レスポンス**
```typescript
{ total_count: number; items: Repository[] }
```

---

## 型定義

```typescript
// src/types/github.ts

export interface Repository {
  id: number
  full_name: string
  description: string | null
  html_url: string
  language: string | null
  stargazers_count: number
  forks_count: number
  open_issues_count: number
  pushed_at: string
  topics: string[]
}

export type SearchMode = 'language' | 'topic' | 'both'

export interface SearchSettings {
  minStars: number
  minForks: number
  minIssues: number      // クライアントサイドフィルター
  commitWithin: number   // 日数。0 = 制限なし
}
```

---

## Zustand Store

```typescript
// src/stores/authStore.ts
interface AuthState {
  user: { login: string; avatar_url: string } | null
  searchMode: SearchMode | null
  setUser: (user: AuthState['user']) => void
  setSearchMode: (mode: SearchMode) => void
}

// src/stores/settingsStore.ts
interface SettingsState {
  settings: SearchSettings
  setSettings: (s: SearchSettings) => void
}
// localStorage に永続化すること（zustand/middleware の persist を使用）
```

---

## 各画面仕様

### LoginPage

- ロゴ表示
- 「GitHubでログイン」ボタン → OAuth リダイレクト
- `/api/auth/me` が 200 なら MainPage へリダイレクト（二重ログイン防止）

### OnboardingPage

`authStore.searchMode === null` のユーザーにのみ表示。

3択カードで検索起点を選択:

| 選択肢 | 説明 |
|--------|------|
| 言語 | プログラミング言語で絞り込み |
| トピック | GitHub トピックで絞り込み |
| 両方 | 言語 + トピックを組み合わせ |

選択後 MainPage へ遷移。

### SettingsPage

#### 活発さプリセット

| プリセット | minStars | minForks | minIssues | commitWithin |
|------------|----------|----------|-----------|--------------|
| 超活発 | 1000 | 100 | 50 | 7 |
| 活発 | 500 | 50 | 20 | 30 |
| 普通 | 100 | 10 | 5 | 90 |

ボタンクリックで全フィルター一括設定。

#### 最終コミットプリセット

`1週間` / `1ヶ月` / `3ヶ月` / `6ヶ月` / `制限なし` のボタン選択。

#### スライダー

| フィルター | min | max | step |
|------------|-----|-----|------|
| Star 最小値 | 0 | 10000 | 100 |
| Fork 最小値 | 0 | 1000 | 10 |
| Issue 最小値 | 0 | 500 | 5 |

現在値をスライダー横にリアルタイム表示。  
「保存」ボタンで settingsStore に反映 → MainPage に戻る。

> Issue 最小値はサーバー側クエリに含めず、レスポンス後クライアントで `open_issues_count >= minIssues` でフィルタリング。

### MainPage

- ヘッダー: ロゴ / アバター+ユーザー名 / 設定アイコン / モード変更リンク
- searchMode に応じて言語・トピック入力欄を表示
- 適用中フィルターをタグ表示（minStars / minForks / commitWithin）
- 検索ボタン or Enter で `/api/github/search` を呼び出し
- 検索結果を RepoCard グリッドで表示（Star数降順）
- 「もっと見る」ボタン → page+1 で追加取得

#### RepoCard

- リポジトリ名（`owner/repo`）→ クリックで GitHub を新規タブで開く
- 説明文
- 言語（言語別カラードット付き）
- トピックタグ（最大4件）
- Star数 / Fork数 / Open Issue数
- 最終コミット相対日時（「3日前」等）

---

## Biome 設定

```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2
  }
}
```

---

## vercel.json

```json
{
  "functions": {
    "api/**/*.ts": {
      "runtime": "edge"
    }
  },
  "rewrites": [
    { "source": "/api/(.*)", "destination": "/api/[[...route]]" }
  ]
}
```

---

## テスト方針

Vitest で以下を優先的にテストすること:

- `src/lib/github.ts` — クエリ構築ロジック (`buildSearchQuery`)
- `src/stores/settingsStore.ts` — プリセット適用・persist 動作
- `api/[[...route]].ts` — Hono ルートのユニットテスト（`hono/testing` 使用）

---

## 実装上の注意

- GitHub Search API レート制限: 認証済みで 30 req/min
- `open_issues_count` フィルタリングはクライアントサイドで実施
- OnboardingPage 表示判定は `authStore.searchMode === null`
- Cookie 署名には `SESSION_SECRET` を使用
- Vercel Edge Functions では Node.js API 非対応 → Web Crypto API を使用
