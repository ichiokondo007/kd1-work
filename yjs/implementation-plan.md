# Yjs 共同編集 実装プラン

## Phase 1: Circle 共同編集 (現在)

### ゴール
2つのブラウザタブで同じ Canvas を開き、Circle の追加・移動・リサイズ・削除がリアルタイムに同期されること。

### 実装ステップ

| Step | 内容 | 状態 |
|------|------|------|
| 1 | `apps/yjs-server` 構築（TypeScript/ESM, y-websocket v1.5.4 ベース再実装） | **完了** |
| 2 | クライアント依存パッケージ追加 (`yjs`, `y-websocket`) | **完了** |
| 3 | `useYjsConnection` hook（WebsocketProvider 接続・Awareness 管理） | **完了** |
| 4 | `useYjsCircleSync` hook（Circle の Fabric ↔ Y.Map バインディング） | **完了** |
| 5 | `/example/canvas-yjs/:id` エディタページ統合 | **完了** |

### クライアント側ファイル構成

```
apps/client/src/
├── features/canvas-yjs/
│   ├── types.ts                        # CollabStatus, YjsCanvasListItem
│   ├── hooks/
│   │   ├── useYjsConnection.ts         # WebsocketProvider 接続管理
│   │   ├── useYjsCircleSync.ts         # Fabric ↔ Y.Map バインディング
│   │   └── useYjsCanvasList.ts         # 一覧取得
│   ├── ui/
│   │   ├── CollabStatusBadge.tsx       # 一覧用ステータスバッジ
│   │   ├── ConnectedUsers.tsx          # Awareness 表示（アバター一覧）
│   │   └── ConnectionStatusBadge.tsx   # 接続状態バッジ
│   ├── domain/
│   │   └── rules.ts                    # ステータス判定（純関数）
│   ├── services/
│   │   └── index.ts                    # Yjs 固有 API（Phase 2）
│   └── index.ts
├── pages/example/
│   ├── canvas-yjs-list.tsx             # 一覧ページ
│   └── canvas-yjs-editor.tsx           # 共同編集エディタページ
```

## Phase 1.6: Rect / 登録 SVG 共同編集（完了）

- `Y.Map("rects")` + `useYjsRectSync` — Rect 標準プロパティを CRDT 同期
- `Y.Map("svgPlacements")` + `useYjsSvgPlacementSync` — `svgAssetKey` + `svgAssetUrl` と変形（A 案）
- `collabRemoteApplyDepthRef` で Circle / Rect / SVG の非同期復元と `object:added` ループを防止
- Mongo persistence: `expandCanvasToYDoc` / `collapseYDocToCanvasJson` が Rect と `type: kd1SvgPlacement` を扱う
- `FabricCanvas.placeSvgFromUrl(url, { key })` で Yjs 用メタを Group に付与

## Phase 2: 永続化 + スケーリング検証

| Step | 内容 |
|------|------|
| 2-1 | MongoDB Persistence 実装 (bindState / writeState) |
| 2-2 | Canvas JSON → Y.Doc 変換、Y.Doc → Canvas JSON 変換 |
| 2-3 | Rect 等の他オブジェクト対応 |
| 2-4 | Docker 化 (Dockerfile.yjs-server, docker-compose 追加) |
| 2-5 | スケーリング方式選定 (y/hub vs Hocuspocus) |
| 2-6 | マルチインスタンス構成構築 |
| 2-7 | 負荷テストクライアント作成 |
| 2-8 | メトリクス収集基盤 (Prometheus + Grafana) |
| 2-9 | ベンチマーク実施 (100Canvas × 10人) |
