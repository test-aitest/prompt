# バックエンド設計書

## 1. アーキテクチャ概要

### システム構成
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Next.js App   │    │   Supabase DB   │    │   OpenAI API    │
│  (Frontend +    │◄──►│   (PostgreSQL)  │    │   (GPT-4o-mini) │
│   API Routes)   │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                       │
        ▼                       ▼
┌─────────────────┐    ┌─────────────────┐
│     Vercel      │    │  Supabase Auth  │
│   (Hosting)     │    │   (GitHub OAuth)│
└─────────────────┘    └─────────────────┘
```

### 技術スタック
- **Runtime**: Node.js 18+
- **Framework**: Next.js 14 (App Router)
- **Database**: Supabase (PostgreSQL)
- **Authentication**: Supabase Auth (GitHub OAuth)
- **External API**: OpenAI GPT-4o-mini
- **Deployment**: Vercel

## 2. データベース設計

### 2.1 テーブル設計

#### users テーブル（Supabase Auth標準）
```sql
-- Supabase Authが自動生成
auth.users (
  id UUID PRIMARY KEY,
  email VARCHAR,
  raw_user_meta_data JSONB,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
)
```

#### prompts テーブル
```sql
CREATE TABLE public.prompts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  original_prompt TEXT NOT NULL,
  optimized_result JSONB NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- インデックス
CREATE INDEX idx_prompts_user_id ON prompts(user_id);
CREATE INDEX idx_prompts_created_at ON prompts(created_at DESC);

-- RLS (Row Level Security)
ALTER TABLE prompts ENABLE ROW LEVEL SECURITY;

-- ポリシー：ユーザーは自分のプロンプトのみアクセス可能
CREATE POLICY "Users can only access their own prompts"
  ON prompts FOR ALL
  USING (auth.uid() = user_id);
```

### 2.2 データ構造

#### optimized_result JSONB 構造
```typescript
interface OptimizedResult {
  purpose: string;
  structure: {
    system: string;
    user: string;
    format: string;
  };
  improvements: {
    title: string;
    content: string;
    type: 'clarity' | 'structure' | 'quality';
  }[];
  metadata: {
    processing_time: number;
    model_used: string;
    tokens_used: number;
  };
}
```

### 2.3 データ管理ポリシー

#### 自動削除機能
```sql
-- 古いプロンプトの自動削除関数
CREATE OR REPLACE FUNCTION cleanup_old_prompts()
RETURNS void AS $$
BEGIN
  -- ユーザーあたり50件を超える古いプロンプトを削除
  DELETE FROM prompts
  WHERE id IN (
    SELECT id FROM (
      SELECT id, ROW_NUMBER() OVER (
        PARTITION BY user_id 
        ORDER BY created_at DESC
      ) as rn
      FROM prompts
    ) ranked
    WHERE rn > 50
  );
END;
$$ LANGUAGE plpgsql;

-- 日次実行のcron設定（Supabase Edge Functions）
```

## 3. API設計

### 3.1 エンドポイント一覧

| Method | Endpoint | 機能 | 認証 |
|--------|----------|------|------|
| POST | `/api/optimize` | プロンプト最適化 | 必須 |
| GET | `/api/history` | 履歴取得 | 必須 |
| DELETE | `/api/history/[id]` | 履歴削除 | 必須 |
| GET | `/api/auth/user` | ユーザー情報取得 | 必須 |

### 3.2 API詳細設計

#### POST /api/optimize
**目的**: プロンプトの最適化実行

**リクエスト**
```typescript
interface OptimizeRequest {
  prompt: string; // 最大2000文字
}
```

**レスポンス**
```typescript
interface OptimizeResponse {
  success: boolean;
  data?: OptimizedResult;
  error?: {
    code: 'RATE_LIMIT' | 'INVALID_INPUT' | 'API_ERROR' | 'AUTH_ERROR';
    message: string;
  };
}
```

**実装例**
```typescript
// app/api/optimize/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';
import { optimizePrompt } from '@/lib/openai';

export async function POST(request: NextRequest) {
  try {
    const supabase = createClient();
    const { data: { user }, error: authError } = await supabase.auth.getUser();
    
    if (authError || !user) {
      return NextResponse.json(
        { success: false, error: { code: 'AUTH_ERROR', message: 'Unauthorized' } },
        { status: 401 }
      );
    }

    const { prompt } = await request.json();
    
    // バリデーション
    if (!prompt || prompt.length > 2000) {
      return NextResponse.json(
        { success: false, error: { code: 'INVALID_INPUT', message: 'Invalid prompt' } },
        { status: 400 }
      );
    }

    // レート制限チェック
    const rateLimit = await checkRateLimit(user.id);
    if (!rateLimit.allowed) {
      return NextResponse.json(
        { success: false, error: { code: 'RATE_LIMIT', message: 'Rate limit exceeded' } },
        { status: 429 }
      );
    }

    // OpenAI API呼び出し
    const optimizedResult = await optimizePrompt(prompt);
    
    // データベースに保存
    const { error: dbError } = await supabase
      .from('prompts')
      .insert({
        user_id: user.id,
        original_prompt: prompt,
        optimized_result: optimizedResult
      });

    if (dbError) throw dbError;

    return NextResponse.json({
      success: true,
      data: optimizedResult
    });

  } catch (error) {
    console.error('Optimize API Error:', error);
    return NextResponse.json(
      { success: false, error: { code: 'API_ERROR', message: 'Internal server error' } },
      { status: 500 }
    );
  }
}
```

#### GET /api/history
**目的**: ユーザーの最適化履歴取得

**クエリパラメータ**
```typescript
interface HistoryQuery {
  limit?: number; // デフォルト: 20, 最大: 50
  offset?: number; // デフォルト: 0
}
```

**レスポンス**
```typescript
interface HistoryResponse {
  success: boolean;
  data?: {
    prompts: PromptHistory[];
    total: number;
    hasMore: boolean;
  };
  error?: ErrorResponse;
}

interface PromptHistory {
  id: string;
  original_prompt: string;
  optimized_result: OptimizedResult;
  created_at: string;
}
```

## 4. 外部API連携

### 4.1 OpenAI API連携

#### 設定
```typescript
// lib/openai.ts
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

export async function optimizePrompt(originalPrompt: string): Promise<OptimizedResult> {
  const systemPrompt = `
あなたは「プロンプト最適化の専門家」です。ユーザーが入力したプロンプトを分析し、以下の手順で最適化してください：

## 分析手順
1. **目的判定**：このプロンプトで何を達成しようとしているか
2. **構造分離**：システム指示・ユーザー質問・期待する応答形式に分離
3. **改善案作成**：3つの異なるアプローチで改善版を提案

## 出力フォーマット
以下のJSON形式で回答してください：
{
  "purpose": "プロンプトの意図・目的",
  "structure": {
    "system": "AIの役割・制約",
    "user": "具体的な質問・要求", 
    "format": "期待する応答スタイル"
  },
  "improvements": [
    {
      "title": "明確化重視",
      "content": "より具体的で明確な指示",
      "type": "clarity"
    },
    {
      "title": "構造化重視", 
      "content": "ステップバイステップの構造",
      "type": "structure"
    },
    {
      "title": "出力品質重視",
      "content": "出力フォーマットを詳細指定",
      "type": "quality"
    }
  ]
}
`;

  const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: originalPrompt }
    ],
    temperature: 0.7,
    max_tokens: 2000,
  });

  const result = JSON.parse(response.choices[0].message.content || '{}');
  
  return {
    ...result,
    metadata: {
      processing_time: Date.now(),
      model_used: 'gpt-4o-mini',
      tokens_used: response.usage?.total_tokens || 0
    }
  };
}
```

### 4.2 エラーハンドリング

#### OpenAI API エラー
```typescript
export async function optimizePrompt(originalPrompt: string): Promise<OptimizedResult> {
  try {
    // API呼び出し
  } catch (error) {
    if (error instanceof OpenAI.APIError) {
      switch (error.status) {
        case 429:
          throw new Error('RATE_LIMIT');
        case 401:
          throw new Error('API_KEY_ERROR');
        default:
          throw new Error('OPENAI_API_ERROR');
      }
    }
    throw error;
  }
}
```

## 5. セキュリティ対策

### 5.1 認証・認可
- **JWT Token**: Supabase Auth標準のJWT使用
- **Row Level Security**: データベースレベルでの認可
- **CORS設定**: 適切なオリジン制限

### 5.2 入力検証
```typescript
// バリデーション関数
export function validatePromptInput(prompt: string): boolean {
  if (!prompt || typeof prompt !== 'string') return false;
  if (prompt.length > 2000) return false;
  if (prompt.trim().length === 0) return false;
  
  // 悪意のあるコンテンツのチェック
  const dangerousPatterns = [
    /<script/i,
    /javascript:/i,
    /on\w+=/i
  ];
  
  return !dangerousPatterns.some(pattern => pattern.test(prompt));
}
```

### 5.3 レート制限
```typescript
// Redis代替としてSupabaseを使用したレート制限
export async function checkRateLimit(userId: string): Promise<{ allowed: boolean; resetTime?: number }> {
  const supabase = createClient();
  
  const thirtySecondsAgo = new Date(Date.now() - 30 * 1000);
  
  const { data, error } = await supabase
    .from('prompts')
    .select('created_at')
    .eq('user_id', userId)
    .gte('created_at', thirtySecondsAgo.toISOString());
  
  if (error) throw error;
  
  return {
    allowed: data.length === 0,
    resetTime: data.length > 0 ? new Date(data[0].created_at).getTime() + 30000 : undefined
  };
}
```

## 6. パフォーマンス最適化

### 6.1 データベース最適化
- **インデックス設定**: user_id, created_at にインデックス
- **クエリ最適化**: 必要なフィールドのみ選択
- **ページネーション**: 履歴表示で適切なページング

### 6.2 API最適化
- **レスポンス圧縮**: gzip圧縮有効化
- **キャッシュ制御**: 適切なCache-Controlヘッダー
- **並列処理**: 可能な処理の並列化

### 6.3 外部API最適化
- **タイムアウト設定**: 15秒タイムアウト
- **リトライ機能**: 3回まで自動リトライ
- **コスト制御**: GPT-4o-mini使用でコスト削減

## 7. 監視・ログ

### 7.1 ログ設計
```typescript
// ログ構造
interface LogEntry {
  timestamp: string;
  level: 'info' | 'warn' | 'error';
  service: string;
  userId?: string;
  action: string;
  details: Record<string, any>;
}

// 使用例
logger.info('prompt_optimized', {
  userId: user.id,
  promptLength: prompt.length,
  processingTime: endTime - startTime,
  tokensUsed: response.usage?.total_tokens
});
```

### 7.2 メトリクス監視
- **API応答時間**: 平均・95パーセンタイル
- **エラー率**: 5分間隔での監視
- **外部API使用量**: OpenAI API使用量・コスト
- **データベース性能**: クエリ実行時間

## 8. 環境変数管理

### 8.1 必要な環境変数
```bash
# OpenAI
OPENAI_API_KEY=sk-xxx

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx
SUPABASE_SERVICE_ROLE_KEY=eyJxxx

# App Config
NEXT_PUBLIC_APP_URL=https://prompt-refinery.vercel.app
```

### 8.2 環境別設定
- **Development**: `.env.local`
- **Production**: Vercel Environment Variables
- **Testing**: `.env.test` 