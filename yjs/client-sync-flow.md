# Yjs Canvas クライアント同期フロー

> 対象: KD1 Canvas アプリケーション — Fabric.js ↔ Y.Doc 双方向同期の詳細フロー
> 対象読者: 本プロジェクトの実装開発者
> 最終更新: 2026-03-23 (useYjsObjectSync ベース — 全オブジェクト対応)

---

## 目次

1. [概要](#1-概要)
2. [関連ファイル一覧](#2-関連ファイル一覧)
3. [ローカル操作 → yjs-server への送信](#3-ローカル操作--yjs-server-への送信)
4. [ブロードキャスト受信 → Canvas への反映](#4-ブロードキャスト受信--canvas-への反映)
5. [ループ防止機構 (collabRemoteDepth)](#5-ループ防止機構-collabremotedepth)
6. [初期同期・復元パス](#6-初期同期復元パス)
7. [フロー全体図](#7-フロー全体図)
8. [補足: 使用していないパターン](#8-補足-使用していないパターン)

---

## 1. 概要

Canvas 上の Fabric.js オブジェクトと Yjs `Y.Map("objects")` の双方向同期は、
**`useYjsObjectSync`** フックが一元的に担当する。

オブジェクト種別を問わず `fabricSnapshot`（`obj.toObject()` の完全 JSON）で統一的に扱い、
Phase 1 の Circle 限定から全オブジェクト対応へ拡張された。

### データ構造

```typescript
// Y.Map("objects") のエントリ
interface ObjectYjsEntry {
  fabricSnapshot: Record<string, unknown>;
}
```

### ID 管理

- `WeakMap<FabricObject, string>` でオブジェクト ↔ UUID を紐付け
- ローカル生成: `crypto.randomUUID()` または Fabric オブジェクトの `yjsId` プロパティから取得
- リモート受信: Y.Map のキー（UUID）を `setObjectId()` で登録

---

## 2. 関連ファイル一覧

### クライアント（apps/client）

| ファイル | 役割 |
| --- | --- |
| `features/canvas-yjs/hooks/useYjsObjectSync.ts` | Fabric.js ↔ Y.Map 双方向バインディング（中核） |
| `features/canvas-yjs/hooks/useYjsConnection.ts` | WebSocket 接続管理 + Awareness セットアップ |
| `features/canvas-yjs/hooks/useYjsCanvasRestore.ts` | Y.Map("meta") からの背景画像・Canvas 名復元 |
| `features/canvas-yjs/hooks/collabRemoteDepth.ts` | リモート適用中フラグ（ループ防止） |
| `features/canvas/ui/FabricCanvas.tsx` | Fabric Canvas 本体（図形追加時に `yjsId` 付与） |
| `features/canvas/fabricRegisterGroupSvgMetadata.ts` | SVG Group の `yjsId` シリアライズ対応 |
| `pages/example/canvas-yjs-editor.tsx` | エディタページ（hooks の組み立て役） |

### サーバ（apps/yjs-server）

| ファイル | 役割 |
| --- | --- |
| `yjs/doc-registry.ts` | Y.Doc シングルトン管理、update/awareness ブロードキャスト |
| `yjs/connection.ts` | WebSocket 接続ハンドリング、Sync/Awareness プロトコル処理 |
| `yjs/server.ts` | HTTP/WebSocket サーバ起動、upgrade ハンドリング |
| `kd1/persistence.ts` | MongoDB → Y.Doc 初期展開（`expandCanvasToYDoc`） |

---

## 3. ローカル操作 → yjs-server への送信

**フロー: Fabric イベント → Y.Map 更新 → WebsocketProvider 自動送信 → yjs-server**

### Step 1: Fabric イベントのキャッチ

`useYjsObjectSync.ts` で 3 つのイベントをリッスンする（198-200 行）:

```typescript
canvas.on("object:modified", handleObjectModified);
canvas.on("object:added", handleObjectAdded);
canvas.on("object:removed", handleObjectRemoved);
```

#### ADD 時 — `handleObjectAdded`（108-114 行）

```typescript
const handleObjectAdded = (e: { target?: FabricObject }) => {
  if (isApplyingRemote(depthRef) || !e.target) return;
  const id = getObjectId(e.target);
  if (!yObjects.has(id)) {
    yObjects.set(id, fabricToYjs(e.target));
  }
};
```

- `isApplyingRemote` ガードでリモート適用中はスキップ
- `yObjects.has(id)` で重複登録を防止
- `fabricToYjs()` が `obj.toObject()` を `fabricSnapshot` として格納

#### UPDATE 時 — `handleObjectModified`（102-106 行）

```typescript
const handleObjectModified = (e: { target?: FabricObject }) => {
  if (isApplyingRemote(depthRef) || !e.target) return;
  const id = getObjectId(e.target);
  yObjects.set(id, fabricToYjs(e.target));
};
```

- ドラッグ/リサイズ/回転完了時に発火
- 既存エントリを上書き（`set` は add or update）

#### DELETE 時 — `handleObjectRemoved`（116-122 行）

```typescript
const handleObjectRemoved = (e: { target?: FabricObject }) => {
  if (isApplyingRemote(depthRef) || !e.target) return;
  const id = getObjectId(e.target);
  if (yObjects.has(id)) {
    yObjects.delete(id);
  }
};
```

### Step 2: Y.Doc → WebSocket 送信（y-websocket 内部）

`useYjsConnection.ts`（38-40 行）で `WebsocketProvider` が Y.Doc にバインドされる:

```typescript
const wsProvider = new WebsocketProvider(buildWsUrl(), canvasId, doc, {
  connect: true,
});
```

**このリポジトリ内に明示的な `conn.send()` はない。**
`y-websocket` の `WebsocketProvider` が内部で `Y.Doc` の `update` イベントを監視し、
差分バイナリを自動的に WebSocket で送信する。

### Step 3: yjs-server での受信・マージ

`connection.ts`（37-42 行）で同期メッセージを処理:

```typescript
case MESSAGE_SYNC:
  encoding.writeVarUint(encoder, MESSAGE_SYNC);
  syncProtocol.readSyncMessage(decoder, encoder, doc, null);
  if (encoding.length(encoder) > 1) {
    conn.send(encoding.toUint8Array(encoder));
  }
```

`syncProtocol.readSyncMessage` がクライアントから来た同期メッセージ（SyncStep2 / Update）を
サーバ側の `WSSharedDoc` にマージする。

### Step 4: 他クライアントへブロードキャスト

マージにより `doc.on("update")` が発火し、全接続に broadcast する。
`doc-registry.ts`（77-83 行）:

```typescript
doc.on("update", (update: Uint8Array, _origin: unknown) => {
  const encoder = encoding.createEncoder();
  encoding.writeVarUint(encoder, MESSAGE_SYNC);
  syncProtocol.writeUpdate(encoder, update);
  const message = encoding.toUint8Array(encoder);
  doc.conns.forEach((_ids, conn) => broadcastToConn(doc, conn, message));
});
```

`broadcastToConn`（88-105 行）が実際の `conn.send(message)` を実行する。

---

## 4. ブロードキャスト受信 → Canvas への反映

**フロー: yjs-server broadcast → WebsocketProvider → Y.Doc ローカル更新 → Y.Map observe → Fabric canvas.add**

### Step 1: 受信・ローカル Y.Doc 更新

`y-websocket` の `WebsocketProvider` がサーバからの sync メッセージを受信し、
ローカルの `Y.Doc` に `applyUpdate` する（ライブラリ内部処理）。

### Step 2: Y.Map の observe コールバック

`useYjsObjectSync.ts`（201 行）で observer を登録:

```typescript
yObjects.observe(observer);
```

`observer`（124-196 行）が Y.Map の変更を検知し、`change.action` に応じて Fabric Canvas を更新する。

#### ADD（新規オブジェクト受信）— 127-151 行

```typescript
case "add": {
  const entry = yObjects.get(key);
  if (!entry) break;
  if (findFabricObjectById(canvas, key)) break;  // 重複チェック
  void (async () => {
    depthRef.current += 1;
    try {
      const obj = await snapshotToFabricObject(entry.fabricSnapshot);
      const c = fabricRef.current?.getCanvas();
      if (!obj || !c) return;
      setObjectId(obj, key);   // WeakMap に Y.Map キー（UUID）を登録
      c.add(obj);
      c.requestRenderAll();
    } finally {
      depthRef.current -= 1;
    }
  })();
  break;
}
```

- `snapshotToFabricObject` は `util.enlivenObjects` で fabricSnapshot から FabricObject を復元
- `setObjectId(obj, key)` でリモートオブジェクトの ID を WeakMap に登録

#### UPDATE（既存オブジェクト更新受信）— 153-179 行

```typescript
case "update": {
  const entry = yObjects.get(key);
  if (!entry) break;
  const existing = findFabricObjectById(canvas, key);
  if (!existing) break;
  void (async () => {
    depthRef.current += 1;
    try {
      const obj = await snapshotToFabricObject(entry.fabricSnapshot);
      const c = fabricRef.current?.getCanvas();
      if (!obj || !c) return;
      setObjectId(obj, key);
      c.remove(existing);   // 既存を削除
      c.add(obj);           // 新インスタンスで置換
      c.requestRenderAll();
    } finally {
      depthRef.current -= 1;
    }
  })();
  break;
}
```

update は **既存オブジェクトを `remove` → 新しいインスタンスを `add`** する置換方式。
プロパティ個別更新ではなく全体置換とすることで、オブジェクト種別を問わず統一的に扱える。

#### DELETE（オブジェクト削除受信）— 181-192 行

```typescript
case "delete": {
  const existing = findFabricObjectById(canvas, key);
  if (existing) {
    depthRef.current += 1;
    try {
      canvas.remove(existing);
      canvas.requestRenderAll();
    } finally {
      depthRef.current -= 1;
    }
  }
  break;
}
```

---

## 5. ループ防止機構 (collabRemoteDepth)

### 問題

双方向バインディングでは以下の無限ループが発生しうる:

```
リモート受信 → canvas.add(obj) → "object:added" 発火 → yObjects.set() → 再送信 → ...
```

### 解決: depthRef カウンタ

`collabRemoteDepth.ts` の `isApplyingRemote`:

```typescript
export function isApplyingRemote(
  depthRef: React.MutableRefObject<number>,
): boolean {
  return depthRef.current > 0;
}
```

- observer 内の処理開始時: `depthRef.current += 1`
- observer 内の処理終了時: `depthRef.current -= 1`（`finally` ブロック）
- Fabric イベントハンドラ冒頭: `isApplyingRemote(depthRef)` が `true` ならスキップ

Phase 1 の `isRemoteRef`（boolean）から **カウンタ方式** に変更された。
理由: observer 内で `async` 処理（`snapshotToFabricObject`）を行うため、
複数の非同期処理が並行する場合に boolean では正しく管理できない。

### ループ断ち切りフロー

```
[ユーザ A がローカルで図形を追加]
  Fabric "object:added" 発火
    → depthRef.current === 0 → yObjects.set() 実行
      → WebSocket でサーバに送信 → 全クライアントにブロードキャスト

[ユーザ B で受信]
  Y.Map observer 発火
    → depthRef.current += 1 (= 1)
      → canvas.add(obj) 実行
        → Fabric "object:added" 発火
          → depthRef.current === 1 → isApplyingRemote = true → スキップ
    → depthRef.current -= 1 (= 0)
```

---

## 6. 初期同期・復元パス

### 6.1 オブジェクト初期同期

`useYjsObjectSync.ts`（203 行, 214-238 行）の `renderYjsObjectsToCanvas`:

```typescript
renderYjsObjectsToCanvas(canvas, yObjects, fabricRef, depthRef);
```

useEffect 末尾で呼ばれ、Y.Map に既存エントリがあれば Fabric Canvas に描画する。
途中参加で SyncStep2 が先に完了していた場合に、observer 登録前のデータを反映する。

```typescript
function renderYjsObjectsToCanvas(...): void {
  if (yObjects.size === 0) return;
  yObjects.forEach((entry, key) => {
    if (findFabricObjectById(canvas, key)) return;  // 重複チェック
    void (async () => {
      depthRef.current += 1;
      try {
        const obj = await snapshotToFabricObject(entry.fabricSnapshot);
        const c = fabricRef.current?.getCanvas();
        if (!obj || !c) return;
        setObjectId(obj, key);
        c.add(obj);
        c.requestRenderAll();
      } finally {
        depthRef.current -= 1;
      }
    })();
  });
}
```

### 6.2 Canvas メタ情報の復元

`useYjsCanvasRestore.ts` が `Y.Map("meta")` から `canvasName` / `backgroundImage` を読み取り、
Canvas の背景を復元する。

### 6.3 サーバ起動時: MongoDB → Y.Doc 展開

`persistence.ts` の `expandCanvasToYDoc`（57-90 行）:

- MongoDB から Canvas データを取得
- `yDoc.getMap("meta")` に `canvasName` / `backgroundImage` をセット
- `yDoc.getMap("objects")` に各オブジェクトの `fabricSnapshot` をセット
- `bindState`（150-163 行）で新規 room 初回接続時に実行される

---

## 7. フロー全体図

```
[ローカル操作]
  Fabric canvas event (object:added / object:modified / object:removed)
    │
    ▼
  handleObjectAdded / handleObjectModified / handleObjectRemoved
  (useYjsObjectSync.ts:102-122)
    │  ※ isApplyingRemote(depthRef) === true ならスキップ
    ▼
  yObjects.set(id, fabricToYjs(obj))    ← Y.Map("objects") 更新
    │
    ▼
  Y.Doc update イベント (y-websocket 内部で自動検知)
    │
    ▼
  WebSocket 送信 (y-websocket WebsocketProvider)
    │
    ▼
  yjs-server messageListener (connection.ts:37-42)
    │  syncProtocol.readSyncMessage → WSSharedDoc マージ
    ▼
  doc.on("update") (doc-registry.ts:77-83)
    │
    ▼
  broadcastToConn → conn.send    ← 全クライアントへ送信


[リモート受信]
  yjs-server broadcast
    │
    ▼
  WebsocketProvider 受信 → ローカル Y.Doc applyUpdate
    │
    ▼
  Y.Map("objects") observe (useYjsObjectSync.ts:124-196)
    │  depthRef.current += 1
    ▼
  case "add":    snapshotToFabricObject → canvas.add(obj)
  case "update": canvas.remove(existing) → canvas.add(newObj)
  case "delete": canvas.remove(existing)
    │  depthRef.current -= 1
    ▼
  canvas.requestRenderAll()
```

---

## 8. 補足: 使用していないパターン

| パターン | 状況 |
| --- | --- |
| `Y.Array` / `observeDeep` | Canvas 同期経路では未使用（`Y.Map` + `observe` のみ） |
| `loadFromJSON` / `fromObject` | Yjs 同期では未使用。復元は `util.enlivenObjects`。`loadFromJSON` は非 Yjs の Canvas エディタで使用 |
| `apps/server` の Yjs 処理 | なし（`apps/yjs-server` が担当） |
| `yMeta.set` (クライアント側) | Canvas メタ情報の書き戻しは現在クライアント側に実装されていない（`meta` の書き込みは主に `persistence.expandCanvasToYDoc`） |
