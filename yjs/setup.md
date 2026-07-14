# Yjs Server セットアップガイド

## ディレクトリ構成

```
apps/yjs-server/
├── package.json          # yjs v13 + ws + y-protocols + lib0
├── tsconfig.json         # ES2022, ESNext, bundler resolution
├── tsup.config.ts        # ESM ビルド (target: node22)
└── src/
    ├── index.ts          # HTTP + WebSocket サーバ エントリポイント
    ├── doc-manager.ts    # Y.Doc シングルトン管理 + WS接続ハンドリング
    └── types.ts          # WSSharedDoc, Persistence 型定義
```

## 依存パッケージ

```json
{
  "dependencies": {
    "lib0": "^0.2.102",
    "ws": "^8.18.0",
    "y-protocols": "^1.0.6",
    "yjs": "^13.6.24"
  },
  "devDependencies": {
    "@types/node": "^22.10.1",
    "@types/ws": "^8.18.0",
    "tsup": "^8.5.1",
    "tsx": "^4.19.2",
    "typescript": "^5.9.3"
  }
}
```

## 起動方法

### 開発

```bash
pnpm --filter yjs-server dev
# → [yjs-server] running at 0.0.0.0:1234
```

### ビルド & 実行

```bash
pnpm --filter yjs-server build
pnpm --filter yjs-server start
```

## 環境変数

| 変数 | デフォルト | 説明 |
|------|-----------|------|
| `HOST` | `0.0.0.0` | バインドアドレス |
| `PORT` | `1234` | ポート番号 |

## ヘルスチェック

```bash
curl http://localhost:1234
```

レスポンス:
```json
{
  "status": "ok",
  "activeDocs": 1,
  "docs": [
    { "name": "canvas-abc123", "connections": 2 }
  ]
}
```

## WebSocket 接続

### エンドポイント

```
ws://localhost:1234/<canvasId>
```

URL パスがそのまま docName (= canvasId) として使用される。

### Vite Proxy 設定

`apps/client/vite.config.ts`:
```typescript
proxy: {
  '/yjs': {
    target: 'ws://localhost:1234',
    ws: true,
    rewriteWsPath: true,
    rewrite: (path) => path.replace(/^\/yjs/, ''),
  },
}
```

クライアントからの接続:
```typescript
// Vite dev server 経由
new WebsocketProvider("ws://localhost:5173/yjs", canvasId, yDoc)

// 直接接続
new WebsocketProvider("ws://localhost:1234", canvasId, yDoc)
```

## doc-manager.ts モジュール公開API

| 関数 | 説明 |
|------|------|
| `setupWSConnection(conn, req, options?)` | WebSocket 接続を処理。Y.Doc の取得/作成、メッセージリスナー、ping/pong を設定 |
| `getDocs()` | アクティブな Y.Doc の Map を返す（ReadonlyMap） |
| `setPersistence(p)` | カスタム永続化レイヤーを設定（Phase 2 で MongoDB 連携に使用） |

### Persistence インターフェース

```typescript
interface Persistence {
  bindState(docName: string, doc: WSSharedDoc): Promise<void>;
  writeState(docName: string, doc: WSSharedDoc): Promise<void>;
}
```

- `bindState`: ドキュメント初回作成時に呼ばれる。永続化層からデータを読み込み Y.Doc に適用
- `writeState`: 全クライアント退出時に呼ばれる。Y.Doc の状態を永続化層に保存

## ログ出力

```
[yjs-server] running at 0.0.0.0:1234        # 起動
[user:joined] doc="canvas-123" connections=1  # 接続
[user:left] doc="canvas-123" connections=0    # 切断
[doc:idle] "canvas-123" — removed from memory # メモリ解放
```

## Docker 対応 (Phase 2)

`docker-compose.yml` に追加予定:

```yaml
yjs-server:
  build:
    context: .
    dockerfile: Dockerfile.yjs-server
  ports:
    - "1234:1234"
  environment:
    HOST: "0.0.0.0"
    PORT: "1234"
  networks:
    - kd1-network
```
