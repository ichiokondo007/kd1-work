# Yjs 公式 API リファレンス（実装用）

調査日: 2026-03-09
対象バージョン: yjs ^13.6.24 / y-websocket v3.0.0 / y-protocols ^1.0.6

---

## 1. y-websocket — WebsocketProvider（クライアント側）

### コンストラクタ

```typescript
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'

const doc = new Y.Doc()
const wsProvider = new WebsocketProvider(
  serverUrl: string,   // 例: 'ws://localhost:1234'
  room: string,        // 例: canvasId
  ydoc: Y.Doc,
  {
    connect: true,           // false なら手動 connect()
    params: {},              // Object<string,string> URL クエリパラメータ（認証トークン等）
    WebSocketPolyfill: WebSocket, // Node.js 環境用（ブラウザでは不要）
    awareness: new Awareness(ydoc),  // 既存 Awareness インスタンスを渡せる
    maxBackoffTime: 2500,    // 再接続の最大バックオフ（ms）
  }
)
```

### プロパティ

| プロパティ | 型 | 説明 |
|-----------|-----|------|
| `wsconnected` | `boolean` | 現在サーバに接続中か |
| `wsconnecting` | `boolean` | 現在接続処理中か |
| `shouldConnect` | `boolean` | false なら再接続しない |
| `bcconnected` | `boolean` | BroadcastChannel 経由で他タブと通信中か |
| `synced` | `boolean` | サーバと同期済みか |
| `params` | `Object<string, string>` | URL パラメータ（動的に更新可、次回接続時に反映） |
| `awareness` | `Awareness` | Awareness インスタンスへのアクセス |

### メソッド

| メソッド | 説明 |
|---------|------|
| `connect()` | WebSocket 接続を確立（`connect: false` で初期化した場合に使用） |
| `disconnect()` | 切断し再接続しない |
| `destroy()` | プロバイダを破棄。切断 + 全イベントハンドラ解除 |

### イベント

```typescript
// 接続状態の変化
wsProvider.on('status', (event: { status: 'connected' | 'connecting' | 'disconnected' }) => {
  console.log(event.status)
})

// サーバとの初期同期完了
wsProvider.on('sync', (isSynced: boolean) => {
  if (isSynced) console.log('Document fully synchronized')
})

// WebSocket 切断イベント転送
wsProvider.on('connection-close', (event: CloseEvent) => { ... })

// WebSocket エラーイベント転送
wsProvider.on('connection-error', (event: Event) => { ... })
```

### 基本使用例

```typescript
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'

const doc = new Y.Doc()
const wsProvider = new WebsocketProvider('ws://localhost:1234', 'my-roomname', doc)

wsProvider.on('status', event => {
  console.log(event.status) // "connected" or "disconnected"
})

// クリーンアップ
window.addEventListener('beforeunload', () => {
  wsProvider.destroy()
})
```

---

## 2. yjs — Y.Doc / Y.Map（共有データ構造）

### Y.Map API

```typescript
const ymap = new Y.Map()

// CRUD
ymap.set(key: string, value: any)
ymap.get(key: string): any
ymap.delete(key: string)
ymap.has(key: string): boolean
ymap.clear()

// プロパティ
ymap.size: number
ymap.parent: Y.AbstractType | null

// イテレーション
ymap.forEach((value, key, map) => { ... })
ymap[Symbol.Iterator]

// シリアライズ
ymap.toJSON(): Object
ymap.clone(): Y.Map
```

### Y.Map の変更監視（observe）

```typescript
const yCircles: Y.Map<CircleProps> = yDoc.getMap("circles")

yCircles.observe((event: Y.YMapEvent, transaction: Y.Transaction) => {
  // event.keys: Map<string, { action, oldValue }>
  event.keys.forEach((change, key) => {
    // change.action: 'add' | 'update' | 'delete'
    // change.oldValue: 変更前の値（update/delete 時）
    const newValue = yCircles.get(key)
  })

  // transaction.origin でローカル/リモートを判別可能
  console.log('local:', transaction.local)
  console.log('origin:', transaction.origin)
})

// ネストされた変更も含めて監視
yCircles.observeDeep((events, transaction) => {
  events.forEach(event => {
    console.log('Changed keys:', event.keys)
  })
})

// 監視解除
yCircles.unobserve(observer)
yCircles.unobserveDeep(deepObserver)
```

### Transaction（バッチ操作）

```typescript
// トランザクション内の複数変更 → observe が1回だけ発火
yDoc.transact(() => {
  yCircles.set('circle-1', props1)
  yCircles.set('circle-2', props2)
}, 'local-operation')  // 第2引数: origin（ローカル/リモート判別用）

// ドキュメントレベルのイベント
yDoc.on('update', (update: Uint8Array, origin, doc, transaction) => {
  console.log('Document updated, size:', update.byteLength)
})
```

### Y.Doc の getMap

```typescript
const doc = new Y.Doc()

// 同じ名前で呼ぶと常に同じ共有インスタンスを返す
const yCircles = doc.getMap("circles")  // Y.Map<CircleProps>
```

---

## 3. y-protocols — Awareness（ユーザ presence）

### 基本操作

```typescript
import * as awarenessProtocol from 'y-protocols/awareness'

// WebsocketProvider から取得（別途インスタンス作成不要）
const awareness = wsProvider.awareness

// 自分の状態を設定
awareness.setLocalState({
  user: {
    userId: 'user-123',
    name: '田中太郎',
    avatarUrl: '/avatars/tanaka.jpg',
    avatarColor: 'blue-500',
  },
  cursor: { x: 340, y: 210 },
})

// 特定フィールドだけ更新（他のフィールドは保持される）
awareness.setLocalStateField('cursor', { x: 500, y: 300 })

// 自分の状態を取得
awareness.getLocalState()
```

### 全ユーザの状態取得

```typescript
const states: Map<number, any> = awareness.getStates()

states.forEach((state, clientID) => {
  if (clientID !== doc.clientID) {
    console.log(`Remote user: ${state.user.name}`)
  }
})
```

### 変更監視

```typescript
// 'change' イベント: 状態が実際に変わったときのみ発火
awareness.on('change', ({ added, updated, removed }) => {
  added.forEach(clientID => {
    const state = awareness.getStates().get(clientID)
    console.log(`User joined: ${state?.user?.name}`)
  })
  updated.forEach(clientID => {
    const state = awareness.getStates().get(clientID)
    console.log(`User updated: cursor at`, state?.cursor)
  })
  removed.forEach(clientID => {
    console.log(`User left: ${clientID}`)
  })
})

// 'update' イベント: すべての awareness 更新で発火（change と同じシグネチャ）
awareness.on('update', ({ added, updated, removed }) => { ... })
```

### 切断時のクリーンアップ

```typescript
window.addEventListener('beforeunload', () => {
  awarenessProtocol.removeAwarenessStates(
    awareness, [doc.clientID], 'window unload'
  )
})
```

### Awareness の手動エンコード/デコード（サーバ側で使用）

```typescript
// エンコード
const update = awarenessProtocol.encodeAwarenessUpdate(
  awareness,
  Array.from(awareness.getStates().keys())
)

// デコード・適用
awarenessProtocol.applyAwarenessUpdate(remoteAwareness, update, 'remote')
```

---

## 4. doc-manager.ts との互換性確認

| 項目 | 公式 @y/websocket-server | doc-manager.ts（自前実装） |
|------|--------------------------|---------------------------|
| Y.Doc シングルトン管理 | docs Map | docs Map |
| Sync プロトコル | y-protocols/sync | y-protocols/sync |
| Awareness プロトコル | y-protocols/awareness | y-protocols/awareness |
| Ping/Pong | 30秒間隔 | 30秒間隔 |
| 全員退出時の cleanup | destroy + persistence | destroy + persistence |
| docName 取得 | URL path から | URL path から |
| 認証 | upgrade 時にトークン検証 | TODO: Phase 2 |

**結論:** `doc-manager.ts` は公式実装と同じプロトコルを使用しており、
`WebsocketProvider` v3 とそのまま互換性あり。

---

## 5. 統合使用例（Canvas 共同編集のパターン）

```typescript
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'

// 1. Y.Doc + Provider 初期化
const yDoc = new Y.Doc()
const wsProvider = new WebsocketProvider(
  'ws://localhost:1234',
  canvasId,
  yDoc,
  { connect: true }
)

// 2. 共有 Map 取得
const yCircles = yDoc.getMap<CircleProps>('circles')

// 3. Awareness 設定
wsProvider.awareness.setLocalState({
  user: { userId, name, avatarUrl, avatarColor }
})

// 4. ローカル操作 → Yjs
// （isRemoteUpdate フラグでループ防止）
let isRemoteUpdate = false

canvas.on('object:modified', (e) => {
  if (isRemoteUpdate) return
  yCircles.set(circleId, fabricToYjs(e.target))
})

// 5. リモート変更 → Fabric
yCircles.observe((event) => {
  isRemoteUpdate = true
  event.keys.forEach((change, key) => {
    switch (change.action) {
      case 'add':    addCircleToCanvas(key, yCircles.get(key)); break
      case 'update': updateCircleOnCanvas(key, yCircles.get(key)); break
      case 'delete': removeCircleFromCanvas(key); break
    }
  })
  isRemoteUpdate = false
})

// 6. 他ユーザの Presence 監視
wsProvider.awareness.on('change', ({ added, updated, removed }) => {
  // UI に接続ユーザ一覧を表示
})

// 7. クリーンアップ
function cleanup() {
  wsProvider.destroy()
  yDoc.destroy()
}
```
