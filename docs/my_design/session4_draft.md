# C社カメラ用SDK 基本設計書 - Session 4

このファイルに、SDKの「エラー設計」「ログ設計」「スレッド・排他制御方針」を書き込んでみましょう。
（※ `【 】`で囲まれた部分はガイドです。適宜書き換えたり削除したりしてください）

---

## 1. エラー設計

### 1.1 エラーコード体系
【SDKが定義するエラーコード（Enumなど）の一覧と、それぞれの意味を記述してください。】

- **エラーコード一覧**:
  - `CSDKError_Success` (0x0): 成功
  - `CSDKError_General` (0x8000): 一般的なエラーカテゴリ
  - `CSDKError_General_InvalidArg` (0x8001): 不正な引数
  - `CSDKError_General_NotInitialized` (0x8002): SDKが初期化されていない
  - `CSDKError_General_DeviceNotFound` (0x8003): 対象のカメラが見つからない
  - `CSDKError_General_InvalidHandle` (0x8005): 無効なカメラハンドル
  - `CSDKError_General_NotSupported` (0x8006): サポートされていない機能
  - `CSDKError_Network` (0x8100): 通信系のエラーカテゴリ
  - `CSDKError_Network_ConnectionFailed` (0x8101): カメラへの接続に失敗
  - `CSDKError_Network_CommunicationError` (0x8102): 通信中のエラー
  - `CSDKError_Process` (0x8200): SDK内部事情のエラーカテゴリ
  - `CSDKError_Process_OutOfMemory` (0x8201): メモリ不足
  - `CSDKError_Process_Unexpected` (0x8202): 予期せぬ内部エラー

### 1.2 非同期エラーの通知方針
【接続中やストリーミング中にバックグラウンドで発生したエラーを、アプリ側にどう通知するか記述してください。】

- 例：`ConnectCamera` 内の非同期処理でエラーが発生した場合は、`ConnectCallback` の引数 `CSDKError` に詳細なエラーコードを格納して通知する。
- 例：ライブプレビュー中の通信切断などの突発的なエラーは、`EventCallback` を通じて `EVENT_TYPE_DEVICE_MISSING` などのイベント種別と共にエラー情報を通知する。

---

## 2. ログ設計

### 2.1 ログレベルと出力基準
【どのようなログレベルを用意し、どういった場面でログを出力するか記述してください。】

- **ERROR**: 動作継続が不可能な致命的エラー（接続失敗、通信途絶、メモリ確保失敗など）
- **WARN**: 動作継続は可能だが、注意が必要な状態（コマンド再送、設定値の範囲外補正など）
- **INFO**: 主要な状態遷移やAPIの開始・終了（Initialize、Connect、StartPreviewなど）
- **DEBUG**: 開発・トラブルシューティング用の詳細情報（コマンド送受信のパケットダンプなど）

### 2.2 ログ出力方式（リダイレクト方針）
【ログの出力先をアプリ側でどう制御できるようにするか記述してください。】

- SDK自体は直接ファイルや標準出力へログを出力せず、アプリが登録した「ログコールバック関数」を経由してログをリダイレクトする設計とする。
- ログコールバックが登録されていない場合は、デフォルトで何も出力しない（またはデバッグビルド時のみ標準エラー出力に出力する）。

```c
// ログコールバックの関数ポインタ定義
typedef void (*CSDKLogCallback)(CSDKLogLevel level, const char* message, void* user_context);

// ログコールバックの登録API
CSDKError SetLogCallback(CSDKLogCallback callback, void* user_context);
```

---

## 3. スレッド・排他制御方針

### 3.1 スレッドセーフ性の保証範囲
【どのAPIがどの程度スレッドセーフ（複数スレッドからの同時呼び出しに対応）であるかを記述してください。】

- **カメラハンドルが異なる場合**: すべてのAPIはスレッド安全であり、複数スレッドから完全に並行して呼び出し可能。
- **同一のカメラハンドルに対する場合**:
  - `GetCameraSetting` / `SetCameraSetting` / `SendCommandCode` などの制御系APIは、SDK内部でミューテックスによる排他制御を行うため、スレッド安全（同時呼び出し時は順次処理される）。
  - `ConnectCamera` / `DisconnectCamera` / `Release` などのライフサイクル管理APIは、同一ハンドルに対して別スレッドから同時に呼び出すことは禁止（未定義動作・二重解放の原因となるため、アプリ側で排他制御するか順序を保証する必要がある）。

### 3.2 SDK内部のスレッドモデル
【SDK内部でどのようなバックグラウンドスレッドを生成・起動するか記述してください。】

- **接続監視・非同期処理スレッド**: `ConnectCamera` を呼んだ際に一時的に生成され、接続シーケンスの非同期実行を担う。
- **画像受信スレッド**: `StartPreview` の呼び出しにより開始され、カメラから画像ストリームをループ受信し、`PreviewCallback` をアプリに通知し続ける。
- **スレッド優先度**: 画像受信スレッドはコマ落ちを防ぐため、高い優先度（High Priority）で動作させる。

---

## 4. 【設計判断】なぜこの堅牢性方針にしたのか？

【「なぜエラーコードをこの体系にしたのか」「なぜログをコールバック方式にしたのか」「スレッドセーフ性の保証範囲をこのように決めた理由」を記述してください。】

- 【エラー設計について】
  - 
- 【ログ設計について】
  - 
- 【スレッド・排他制御方針について】
  - 
