# C社カメラ用SDK 基本設計書 - Session 2

このファイルに、SDKの「API一覧（クラス・関数定義）」と「各APIの入出力詳細」を書き込んでみましょう。
（※ `【 】`で囲まれた部分はガイドです。適宜書き換えたり削除したりしてください。擬似コードはC++, C#, C, Pythonなど、あなたが想定する任意の言語で構いません）

---

## 1. API一覧（クラス・関数定義）

【SDKが提供するクラスや関数の定義を記述してください。】

### 1.1 クラス構成 / インターフェース定義
【クラス構成や主なメソッドのプロトタイプを擬似コード等で定義します。】

- 他言語から扱えるようにC言語でIFを提供する
- 内部的にはC++で実装する

```c
// SDK本体の内部初期化
CSDKError Initialize();
// SDK本体の状態を開放
CSDKError Release();

// カメラデバイスの列挙
// *camera_count: 検出したカメラ数を返す
// **cameras: カメラデバイスの情報を格納した配列を返す
CSDKError EnumCameras(/*in/out*/ size_t* camera_count,
                     /*out*/ CameraDeviceInfo** cameras);
// 列挙で確保したメモリを解放する
CSDKError ReleaseEnumCameras(CameraDeviceInfo* cameras);

// カメラデバイスへの接続
// @brief カメラの接続を確立する。
// @param camera_info カメラデバイスの情報。
// @param connectCallback カメラデバイスの接続完了あるいは失敗時に呼び出されるコールバック関数ポインタ
// @param camera_handle カメラデバイスのハンドル
// @return CSDKError_Success: 成功
//         CSDKError_InvalidArg: 引数が不正な場合
//         CSDKError_Network_DeviceError: カメラの接続確立に失敗した場合
CSDKError ConnectCamera(/*in*/ const CameraDeviceInfo* camera_info,
                        /*in*/ ConnectCallback connectCallback,
                        /*in*/ EventCallback eventCallback,
                        /*in*/ void* user_context);
// カメラデバイスへの切断
// @brief カメラの接続を切断する。
// 要求自体はすぐに返るが、関連する切断イベントはConnectCameraのEventCallbackから非同期に通知される。
// @param camera_handle カメラデバイスのハンドル
// @return CSDKError_Success: 成功
//         CSDKError_InvalidArg: 引数が不正な場合
//         CSDKError_Network_DeviceError: カメラの切断に失敗した場合
CSDKError DisconnectCamera(/*in*/ CameraHandle camera_handle);

// カメラデバイス設定
// @brief カメラの設定を取得する。
// @param camera_handle カメラデバイスのハンドル
// @param property 設定値の構造体
// @return CSDKError_Success: 成功
//         CSDKError_InvalidArg: 引数が不正な場合
//         CSDKError_Network_DeviceError: カメラの設定取得に失敗した場合
CSDKError GetCameraSetting(/*in*/ CameraHandle camera_handle,
                           /*in/out*/ CProperty* property);
// カメラデバイス設定
// @brief カメラの設定を設定する。
// @param camera_handle カメラデバイスのハンドル
// @param property_count 設定項目の数
// @param property 設定値の構造体
// @return CSDKError_Success: 成功
//         CSDKError_InvalidArg: 引数が不正な場合
//         CSDKError_Network_DeviceError: カメラの設定設定に失敗した場合
CSDKError SetCameraSetting(/*in*/ CameraHandle camera_handle,
                            /*in*/ size_t property_count,
                            /*in*/ const CProperty* property);

// 操作コード送信
// @brief カメラに操作コードを送信する。
// @param camera_handle カメラデバイスのハンドル
// @param command_code 操作コード
// @param command_value 操作値
// @return CSDKError_Success: 成功
//         CSDKError_InvalidArg: 引数が不正な場合
//         CSDKError_Network_DeviceError: カメラの操作に失敗した場合
CSDKError SendCommandCode(/*in*/ CameraHandle camera_handle,
                            /*in*/ CCommandCode command_code,
                            /*in*/ CCommandValueUnion* command_value);

// ライブプレビュー開始
CSDKError StartPreview(/*in*/ CameraHandle camera_handle,
                         /*in*/ PreviewCallback previewCallback,
                         /*in*/ void* user_context);
// ライブプレビュー終了
CSDKError StopPreview(/*in*/ CameraHandle camera_handle);
```

---

## 2. API入出力詳細

【主要なAPIについて、引数や戻り値、発生しうるエラーなどの詳細を定義します。】

### 2.1 カメラ検出
* **API名**: `EnumCameras`
* **引数**: 
  * `out size_t* camera_count`: 検出したカメラ数を格納するポインタ
  * `out CameraDeviceInfo** cameras`: 検出したカメラ情報を格納したポインタ
* **戻り値**: `CSDKError`
* **説明**: PCに物理的、またはネットワーク経由で接続されている全てのC社製カメラを自動検出し、その基本情報（シリアル番号、モデル名、接続タイプなど）を返します。

### 2.2 カメラ接続
* **API名**: `ConnectCamera`
* **引数**:
  * `in const CameraDeviceInfo* camera_info`: カメラデバイスの情報
  * `in ConnectCallback connectCallback`: カメラデバイスの接続完了あるいは失敗時に呼び出されるコールバック関数ポインタ
  * `in EventCallback eventCallback`: カメラデバイスのイベントコールバック関数ポインタ
  * `in void* user_context`: ユーザーコンテキスト
* **戻り値**: `CSDKError`
* **説明**: 指定された情報のカメラとの接続を確立し、制御ハンドルを生成します。

### 2.3 カメラデバイス設定
* **API名**: `GetCameraSetting`
* **引数**: 
  * `in CameraHandle camera_handle`: カメラデバイスのハンドル
  * `in/out CProperty* property`: カメラデバイスの設定
* **戻り値**: `CSDKError`
* **説明**: 指定されたカメラデバイスの設定を取得します。

### 2.4 カメラデバイス設定
* **API名**: `SetCameraSetting`
* **引数**: 
  * `in CameraHandle camera_handle`: カメラデバイスのハンドル
  * `in size_t property_count`: 設定項目の数
  * `in const CProperty* property`: カメラデバイスの設定
* **戻り値**: `CSDKError`
* **説明**: 指定されたカメラデバイスの設定を設定します。

### 2.5 操作コード送信
* **API名**: `SendCommandCode`
* **引数**:
  * `in CameraHandle camera_handle`: カメラデバイスのハンドル
  * `in CCommandCode command_code`: 操作コード
  * `in CCommandValueUnion* command_value`: 操作値
* **戻り値**: `CSDKError`
* **説明**: 指定されたカメラデバイスに操作コードを送信します。

### 2.6 ライブプレビュー開始
* **API名**: `StartPreview`
* **引数**:
  * `in CameraHandle camera_handle`: カメラデバイスのハンドル
  * `in PreviewCallback previewCallback`: カメラデバイスのプレビューコールバック関数ポインタ
  * `in void* user_context`: ユーザーコンテキスト
* **戻り値**: `CSDKError`
* **説明**: 指定されたカメラデバイスのライブプレビューを開始します。

### 2.7 ライブプレビュー終了
* **API名**: `StopPreview`
* **引数**:
  * `in CameraHandle camera_handle`: カメラデバイスのハンドル
* **戻り値**: `CSDKError`
* **説明**: 指定されたカメラデバイスのライブプレビューを終了します。

---

## 3. 主要なデータ構造

【APIでやり取りされる構造体や列挙型（Enum）などを定義します。】

### 3.1 `CSDKError` (Enum)
* **CSDKError_Success**: 成功
* **CSDKError_InvalidArg**: 引数が不正な場合
* **CSDKError_Network_DeviceError**: デバイスエラー

### 3.2 `ConnectCallback` (関数ポインタ)
* デバイスの接続完了あるいは失敗時に呼び出される関数
* 引数:
  * `CSDKError error`: コールバックを呼び出した原因
  * `CameraHandle camera_handle`: カメラデバイスのハンドル
  * `void* user_context`: ユーザーコンテキスト

### 3.3 `CameraHandle` (型)
* カメラデバイスの識別に使用するハンドル
* 内部的にはポインタとして扱う

### 3.4 `CProperty` (構造体)
* カメラデバイスの設定を表す構造体
* 設定コード (`CPropertyCode` : Enum)
* 設定値 (`CPropertyValue` : Union)
  * 設定コードによって適切な値
  * SDK内で設定値のサイズを管理することで、メモリの確保の必要性をなくす

### 3.5 `CameraDeviceInfo` (構造体)
* シリアル番号 (`char[64]`)
* モデル名 (`char[128]`)
* 接続インターフェース (`InterfaceType` : USB, Ether, etc.)
  * `INTERFACE_TYPE_USB`
  * `INTERFACE_TYPE_ETHER`

### 3.6 `CCommandCode` (Enum)
* カメラに送信する操作コード

### 3.7 `CCommandValueUnion` (Union)
* カメラに送信する操作値
* CCommandCodeに対応する値を格納する
* サイズが可変なため、適切なメモリ確保のコードを提供し、それを呼び出す関数を用意する

### 3.8 `PreviewCallback` (関数ポインタ)
* ライブプレビューの画像フレームを受け取るコールバック関数
* 引数:
  * `CameraHandle camera_handle`: カメラデバイスのハンドル
  * `CLivePreviewData* live_preview_data`: ライブプレビューデータ
  * `void* user_context`: ユーザーコンテキスト

### 3.9 `EventCallback` (関数ポインタ)
* カメラデバイスのイベントを受け取るコールバック関数
* 引数:
  * `CameraHandle camera_handle`: カメラデバイスのハンドル
  * `CEventData* event_data`: イベントデータ
  * `void* user_context`: ユーザーコンテキスト

### 3.10 `CEventData` (構造体)
* カメラデバイスのイベントデータ
* イベント種別 (`CEventType` : Enum)
  * `EVENT_TYPE_DEVICE_MISSING`: カメラデバイスの通信エラーが発生
  * `EVENT_TYPE_DEVICE_CONNECTED`: カメラデバイスの接続
* イベントパラメータ (`CEventParam` : Union)
  * イベント種別によって適切な値
  * SDK内でイベントパラメータのサイズを管理することで、メモリの確保の必要性をなくす

### 3.11 `CLivePreviewData` (構造体)
* ライブプレビューデータ
* 幅 (`uint32_t`)
* 高さ (`uint32_t`)
* データバッファ (`uint8_t*`)
* データサイズ (`size_t`)
* タイムスタンプ (`uint64_t`)
* フレーム情報 (`CFrameInfo`)

---

## 4. 【設計判断】なぜこのAPI単位・粒度にしたのか？

【「なぜこのクラス構成にしたのか」「なぜこのパラメータ設定方式やプレビュー通知方式を選んだのか」の理由を記述してください。】

* **記述例**:
  * `CameraManager` と `Camera` クラスを分離した理由は、カメラの「検出・選択」という全体管理の役割と、個別の「カメラ制御」の役割を明確に分けるため。これにより、複数カメラの管理に対応しやすくする。
  * プレビューをコールバック方式にした理由は、画像フレームが届いた瞬間にアプリ側で即座に処理ができるようにし、遅延（レイテンシ）を最小限に抑えるため。

* 全て関数にしたのは多言語から使いやすいC言語のABIにしたい為
* Init/Releaseを設けたのは、SDK内部に情報や処理を確保する為
* EnumCamerasを設けたのは、複数台のカメラをカメラ操作とは別で行いたい為
* ConnectCameraの結果通知をコールバックにしたのは、通信によっては時間がかかる為
* カメラデバイス設定の設定値を構造体にしたのは、Set/Getで共用で使えるようにしたい為
