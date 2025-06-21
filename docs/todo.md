# 実装Todo

## 🎯 8時間実装計画

### Phase 1: セットアップ（30分）
- [ ] Next.js 14プロジェクト初期化
- [ ] shadcn/ui + TailwindCSS セットアップ
- [ ] 基本フォルダ構成作成
- [ ] 環境変数設定（.env.local）
- [ ] package.jsonの依存関係追加

### Phase 2: 認証・DB設定（40分）
- [ ] Supabase クライアント設定
- [ ] GitHub OAuth設定
- [ ] データベーステーブル作成
- [ ] RLS（Row Level Security）設定
- [ ] 認証ミドルウェア実装

### Phase 3: GPT変換API（60分）
- [ ] OpenAI API クライアント作成
- [ ] `/api/optimize` エンドポイント実装
- [ ] プロンプト最適化ロジック実装
- [ ] エラーハンドリング
- [ ] レート制限機能

### Phase 4: メインUI（90分）
- [ ] レイアウトコンポーネント作成
- [ ] ヘッダーコンポーネント
- [ ] チャット表示コンポーネント
- [ ] 入力フォームコンポーネント
- [ ] 履歴サイドバー
- [ ] コピー機能実装
- [ ] ローディング状態
- [ ] エラー状態表示

### Phase 5: デプロイ・QA（40分）
- [ ] Vercel設定
- [ ] 環境変数設定（本番）
- [ ] 動作確認・テスト
- [ ] バグ修正

## 📋 詳細実装Todo

### 基本セットアップ
```bash
# 必要なコマンド
pnpm create next-app@latest gpt-app --typescript --tailwind --app
cd gpt-app
pnpm add @supabase/supabase-js @supabase/auth-helpers-nextjs
pnpm add openai
pnpm add @radix-ui/react-dialog @radix-ui/react-button
pnpm add lucide-react
```

### フォルダ構成
```
app/
├── globals.css
├── layout.tsx
├── page.tsx
├── login/
│   └── page.tsx
├── api/
│   ├── optimize/
│   │   └── route.ts
│   └── history/
│       └── route.ts
lib/
├── supabase/
│   ├── client.ts
│   └── server.ts
├── openai.ts
└── utils.ts
components/
├── Header.tsx
├── ChatMessage.tsx
├── PromptInput.tsx
├── HistorySidebar.tsx
├── OptimizedResult.tsx
└── ui/
    ├── button.tsx
    ├── input.tsx
    └── dialog.tsx
```

### データベース設定
```sql
-- Supabaseで実行
CREATE TABLE public.prompts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  original_prompt TEXT NOT NULL,
  optimized_result JSONB NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_prompts_user_id ON prompts(user_id);
CREATE INDEX idx_prompts_created_at ON prompts(created_at DESC);

ALTER TABLE prompts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can only access their own prompts"
  ON prompts FOR ALL
  USING (auth.uid() = user_id);
```

### 環境変数（.env.local）
```bash
OPENAI_API_KEY=sk-xxx
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx
SUPABASE_SERVICE_ROLE_KEY=eyJxxx
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

## 🔧 コンポーネント実装チェックリスト

### app/layout.tsx
- [ ] Supabase Auth Provider設定
- [ ] 基本HTML構造
- [ ] フォント設定

### app/page.tsx
- [ ] 認証チェック
- [ ] メインチャット画面
- [ ] 状態管理（useState）
- [ ] API呼び出し処理

### app/login/page.tsx
- [ ] GitHub OAuth ボタン
- [ ] 認証後リダイレクト
- [ ] シンプルなデザイン

### components/Header.tsx
- [ ] ロゴ・タイトル表示
- [ ] ユーザー情報表示
- [ ] ログアウト機能
- [ ] 履歴ボタン

### components/PromptInput.tsx
- [ ] textarea フォーム
- [ ] 文字数カウンター（2000文字制限）
- [ ] 送信ボタン
- [ ] ローディング状態

### components/ChatMessage.tsx
- [ ] ユーザー・AI メッセージ表示
- [ ] タイムスタンプ
- [ ] コピーボタン

### components/OptimizedResult.tsx
- [ ] 目的分析表示
- [ ] 構造分離表示
- [ ] 改善案3つ表示
- [ ] 各改善案のコピー機能

### components/HistorySidebar.tsx
- [ ] 履歴一覧表示
- [ ] 日付グループ化
- [ ] 履歴選択機能
- [ ] モバイル対応（モーダル）

## 🚀 API実装チェックリスト

### /api/optimize/route.ts
- [ ] POST メソッド実装
- [ ] 認証チェック
- [ ] 入力バリデーション
- [ ] レート制限チェック
- [ ] OpenAI API呼び出し
- [ ] データベース保存
- [ ] エラーハンドリング

### /api/history/route.ts
- [ ] GET メソッド実装
- [ ] 認証チェック
- [ ] ページネーション
- [ ] データ取得・整形

### lib/openai.ts
- [ ] OpenAI クライアント設定
- [ ] プロンプト最適化関数
- [ ] システムプロンプト定義
- [ ] レスポンス処理
- [ ] エラーハンドリング

### lib/supabase/client.ts & server.ts
- [ ] Supabase クライアント設定
- [ ] 認証ヘルパー関数
- [ ] サーバーサイド設定

## 🎨 UI/UX実装チェックリスト

### デザイン
- [ ] ChatGPT風カラーテーマ
- [ ] レスポンシブデザイン
- [ ] アニメーション（ふわっと表示）
- [ ] アイコン（lucide-react）

### 状態管理
- [ ] ローディング状態
- [ ] エラー状態
- [ ] 空の状態（初回）
- [ ] 成功状態

### インタラクション
- [ ] Enter キーで送信
- [ ] コピー成功通知
- [ ] エラー表示
- [ ] レート制限表示

## 🔍 テスト・QA チェックリスト

### 機能テスト
- [ ] ログイン・ログアウト
- [ ] プロンプト送信・受信
- [ ] 履歴表示・選択
- [ ] コピー機能
- [ ] レート制限動作

### エラーハンドリング
- [ ] 長すぎるプロンプト
- [ ] 空のプロンプト
- [ ] API エラー
- [ ] 認証エラー
- [ ] ネットワークエラー

### レスポンシブ
- [ ] モバイル表示
- [ ] タブレット表示
- [ ] デスクトップ表示

### パフォーマンス
- [ ] 初回読み込み速度
- [ ] API応答時間
- [ ] 大量履歴の表示

## 🚀 デプロイチェックリスト

### Vercel設定
- [ ] GitHub連携
- [ ] 環境変数設定
- [ ] ドメイン設定
- [ ] ビルド確認

### 最終確認
- [ ] 本番環境での動作確認
- [ ] エラーログ確認
- [ ] パフォーマンス測定
- [ ] セキュリティチェック

## ⚠️ 注意事項・制約

### 技術制約
- リクエスト間隔: 30秒制限
- 入力文字数: 2000文字制限
- 履歴保存: 50件制限
- OpenAI API: GPT-4o-mini使用

### 実装順序
1. 認証なしでの基本機能実装
2. 認証機能追加
3. 履歴機能実装
4. エラーハンドリング強化
5. UI/UX改善

### 時間管理
- 各フェーズの時間厳守
- 必要最小限の機能に集中
- デバッグ時間を考慮
- 最後に必ずテスト時間を確保 