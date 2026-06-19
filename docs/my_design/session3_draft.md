# C社カメラ用SDK 基本設計書 - Session 3

このファイルに、SDKの「シーケンス図」と「状態遷移図」を書き込んでみましょう。
（※ `【 】`で囲まれた部分はガイドです。適宜書き換えたり削除したりしてください。図の作成にはMermaid記法を推奨しますが、難しければ箇条書きテキストでも構いません）

---

## 1. シーケンス図

【主要なユースケースのシーケンスを定義してください。最低限、「カメラ接続（非同期）」と「プレビュー開始〜フレーム取得〜プレビュー終了」の流れは描き出しましょう。】

### 1.1 カメラ接続シーケンス（正常系・非同期）
【`ConnectCamera` の呼び出しから、コールバックが通知され、ハンドルが有効になるまでの流れを記述してください。】

```mermaid
%% 【ここに接続のシーケンス図を記述してください。以下はテンプレート例です】
sequenceDiagram
    actor App as アプリ
    participant SDK as SDK
    participant Cam as カメラ
    App->>SDK: ConnectCamera(...)
    SDK->>SDK: カメラ用クラスのインスタンス生成
    SDK-->>App: CSDKError_Success (受付が成功したことを返す)
    SDK->>Cam: カメラにPTP通信開始要求
    Cam->>SDK: カメラ初期化完了通知
    SDK->>SDK: handle <- カメラ用クラスのインスタンスのアドレス値
    SDK->>App: ConnectCallback(CSDKError_Success, handle, context)
```

### 1.2 ライブプレビュー開始〜画像取得〜終了シーケンス（正常系）
【`StartPreview` の呼び出し、コールバックでの複数回の画像受信、そして `StopPreview` による終了までの流れを記述してください。】

```mermaid
sequenceDiagram
  actor App as アプリ
  participant SDK as SDK
  participant Cam as カメラ
  App->>SDK: StartPreview(handle)
  SDK->>SDK: カメラインスタンスをプレビュー取得状態に変更
  SDK-->>App: CSDKError_Success (受付が成功したことを返す)
  loop プレビュー中
    SDK->>Cam: プレビュー取得要求
    Cam->>SDK: プレビューデータ
    SDK->>App: PreviewImageCallback(imageData, frameType, timestamp)
  end
  App->>SDK: StopPreview(handle)
  SDK->>SDK: カメラインスタンスのプレビュー取得状態を解除
    
```

### 1.3 プレビュー中の突発的な切断シーケンス（異常系）
【プレビュー中にカメラが物理的に抜けた（UC-07）場合、SDKがそれをどう検知し、アプリのコールバックやAPIを通じてどのようにエラー処理されるかを記述してください。】

```mermaid
sequenceDiagram
  actor App as アプリ
  participant SDK as SDK
  participant Cam as カメラ
  App->>SDK: StartPreview(handle)
  SDK->>SDK: カメラインスタンスをプレビュー取得状態に変更
  SDK-->>App: CSDKError_Success (受付が成功したことを返す)
  loop プレビュー中
    SDK->>Cam: プレビュー取得要求
    Cam->>SDK: プレビューデータ
    SDK->>App: PreviewImageCallback(imageData, frameType, timestamp)
  end
  Note over Cam: カメラの切断
  SDK->>SDK: カメラデバイスの切断を判定
  SDK->>App: EventCallback(CEventData{eventType=EVENT_TYPE_DEVICE_MISSING, eventParam=...}, handle, user_context)
```

---

## 2. 状態遷移図

【SDKまたは接続された個別カメラの「状態遷移」を定義してください。】

### 2.1 SDK/カメラ接続状態の定義
【どのような状態が存在するか、一覧を記述します。】
- **Uninitialized**: SDK初期化前
- **Initialized**: SDK初期化後、カメラ未接続
- **Connecting**: カメラ接続中
- **Connected**: カメラ接続完了
- **Reconnecting**: カメラ接続切断後の再接続中
- **Reconnected**: カメラの再接続完了
- **Disconnected**: カメラの切断完了

### 2.2 状態遷移図
【Mermaid等の記述を用いて、どのイベント（API呼び出しやハードウェア割り込み）によってどの状態へ遷移するかを定義してください。】

```mermaid
%% 【ここに状態遷移図を記述してください】
stateDiagram-v2
    [*] --> Uninitialized
    Uninitialized --> Initialized : InitSDK()
    Initialized --> Connecting : ConnectCamera(handle)
    Connecting --> Connected : ConnectCallback(CSDKError_Success, handle, context)
    Connected --> Disconnected : DisconnectCamera(handle)
    Disconnected --> Initialized : 
    Reconnecting --> Reconnected : ConnectCallback(CSDKError_Success, handle, context)
    Connected --> Reconnecting : カメラと通信エラー(一定時間待機後再接続を試みる)
    Connected --> Disconnected : カメラと通信エラー(再接続設定がOFF)
    Reconnecting --> Disconnected : タイムアウト
    Reconnected --> Disconnected : DisconnectCamera(handle)
```

---

## 3. 【設計判断】なぜこの動的挙動・状態遷移にしたのか？

【「なぜ接続を非同期処理にしたのか」「なぜこの状態定義にしたのか」「異常切断時にどのようなリカバリ遷移にするか」の理由を記述してください。】

* **記述例**:
  * カメラの接続に最大数秒かかる可能性があるため、GUIアプリがフリーズするのを防ぐために `ConnectCamera` を非同期とし、完了はコールバックで通知する設計にした。
  * プレビュー中にエラーが発生した場合は、一度 `Disconnected`（初期状態）まで強制的に状態を引き戻し、アプリ側にリソースの再解放と再接続を要求するシンプルな遷移を採用した。これにより、SDK内部の複雑な再試行ロジックを排除し、堅牢性を高めた。

- 【同期/非同期の設計判断理由について】
  - ConnectCameraを非同期にしたのは処理に時間がかかり、アプリ側がフリーズしてしまう為。コールバックで通知するようにした。
  - StartPreviewを非同期にしたのは画像の取得が連続的なもので一回の呼び出しではない為。
  - DisconnectCameraは非同期でイベントを返すが、その理由は接続と同じで処理に時間がかかるかもしれない為。
- 【状態遷移とエラーリカバリの設計判断理由について】
  - カメラの操作とは別でInit/Releaseを設けたのは、カメラの操作に関連してスレッドプールの立ち上げといった関連リソースの確保/解放を行いたい為。将来的な関連リソースの増加にも対応が可能になる点もある。
  - Connecting/Connectedで別の状態にしているのは、Connectedだけだと接続ができてない状態を適切に管理できない為。
  - カメラは何かしらの接続(USB,Ether,Wifi)で接続されており、必ずしも接続が安定している訳ではない為、再接続が必要なのでReconnecting/Reconnectedを用意した。
  - Connectedから直接Disconnectedにつながるのは、再接続シーケンスを設けずに通信エラーから再度接続するかどうかをアプリに委ねられるようにする為。
