# C社カメラ用SDK 基本設計書 - Session 2 (API・インターフェース設計)

本ドキュメントは、C社カメラ用SDKが外部アプリケーションに提供するAPI（インターフェース）のシグネチャ、主要なデータ構造、および各APIの入出力詳細を定義します。

本SDKは、多様な言語（C#, Python, Unity等）からのバインディング作成や動的ロード（DLL/so）を容易にし、安定したABI（Application Binary Interface）を維持するため、**CリンケージのAPI (C-style API)** として設計されています。

---

## 1. API一覧（クラス・関数定義）

### 1.1 基本型定義・コールバック定義

```c
#include <stdint.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

// 無効なハンドル値の定義
#define CSDK_INVALID_HANDLE ((CameraHandle)0)

// カメラを識別するための不透明なハンドル
// ユーザーにポインタであることを意識させないよう、ポインタ幅と同じサイズを持つ64ビット無符号整数型として定義する
typedef uint64_t CameraHandle;

// SDKエラーコードの型定義 (詳細なエラーコードはデータ構造セクションを参照)
typedef int32_t CSDKError;

// 各種データ構造の前方宣言
typedef struct CameraDeviceInfo CameraDeviceInfo;
typedef struct CProperty CProperty;
typedef union CCommandValueUnion CCommandValueUnion;
typedef struct CEventData CEventData;
typedef struct CLivePreviewData CLivePreviewData;
typedef struct CameraContentInfo CameraContentInfo;

// 接続通知コールバック
// @param error 接続処理の結果コード
// @param camera_handle 接続されたカメラのハンドル
// @param user_context ユーザーが指定した任意のコンテキスト
typedef void (*ConnectCallback)(CSDKError error, CameraHandle camera_handle, void* user_context);

// イベント通知コールバック（突発的な切断や各種ステータス変化など）
// @param camera_handle 対象カメラのハンドル
// @param event_data イベント情報構造体のポインタ
// @param user_context ユーザーが指定した任意のコンテキスト
typedef void (*EventCallback)(CameraHandle camera_handle, const CEventData* event_data, void* user_context);

// ライブプレビューフレーム通知コールバック
// @param camera_handle 対象カメラのハンドル
// @param live_preview_data 受信した画像フレームデータ
// @param user_context ユーザーが指定した任意のコンテキスト
typedef void (*PreviewCallback)(CameraHandle camera_handle, const CLivePreviewData* live_preview_data, void* user_context);

// コンテンツダウンロード進捗・完了通知コールバック
// @param error ダウンロード処理の結果コード
// @param camera_handle 対象カメラのハンドル
// @param downloaded_bytes これまでにダウンロードしたバイト数
// @param total_bytes ファイルの総バイト数
// @param user_context ユーザーが指定した任意のコンテキスト
typedef void (*ContentDownloadCallback)(CSDKError error, CameraHandle camera_handle, size_t downloaded_bytes, size_t total_bytes, void* user_context);
```

### 1.2 SDKライフサイクルおよびカメラ検出・管理

```c
// SDK全体の初期化
// SDKを使用する前に必ず一度呼び出す必要があります。内部処理スレッドや共通リソースの準備を行います。
CSDKError Initialize();

// SDK全体の解放
// SDKの使用を終了する際に呼び出します。確保した全てのリソースを破棄します。
CSDKError Release();

// カメラデバイスの列挙
// @param camera_count 検出されたカメラ台数を受け取る変数へのポインタ (out)
// @param cameras 検出されたカメラ情報の配列の先頭ポインタを受け取る変数へのポインタ (out)
// ※内部でメモリが自動確保されるため、使用後は必ず ReleaseEnumCameras で解放してください。
CSDKError EnumCameras(size_t* camera_count, CameraDeviceInfo** cameras);

// 列挙で確保したメモリの解放
// @param cameras EnumCamerasで取得した配列ポインタ
CSDKError ReleaseEnumCameras(CameraDeviceInfo* cameras);
```

### 1.3 カメラ接続・切断

```c
// カメラデバイスへの接続（非同期）
// @param camera_info 接続対象のカメラ情報
// @param connectCallback 接続完了または失敗時に呼び出されるコールバック
// @param eventCallback 接続後に発生するイベント（切断など）を受け取るコールバック
// @param user_context コールバックに渡されるユーザー任意のコンテキスト
CSDKError ConnectCamera(const CameraDeviceInfo* camera_info,
                        ConnectCallback connectCallback,
                        EventCallback eventCallback,
                        void* user_context);

// カメラデバイスからの切断（非同期）
// @param camera_handle カメラハンドル
CSDKError DisconnectCamera(CameraHandle camera_handle);
```

### 1.4 カメラ設定（プロパティ）制御

```c
// カメラ設定の取得
// @param camera_handle カメラハンドル
// @param property 取得したいプロパティ情報構造体のポインタ (in/out)
//                 (in: property->code に取得対象のコードをセットし、out: property->value に値が格納される)
CSDKError GetCameraSetting(CameraHandle camera_handle, CProperty* property);

// カメラ設定の適用 (複数一括設定)
// @param camera_handle カメラハンドル
// @param property_count 設定するプロパティの件数
// @param properties 設定するプロパティ構造体の配列
CSDKError SetCameraSetting(CameraHandle camera_handle, size_t property_count, const CProperty* properties);
```

### 1.5 ライブプレビュー制御

```c
// ライブプレビューの開始（非同期ストリーミング）
// @param camera_handle カメラハンドル
// @param previewCallback 新しい画像フレームを受信した際に呼び出されるコールバック
// @param user_context コールバックに渡されるユーザー任意のコンテキスト
CSDKError StartPreview(CameraHandle camera_handle, PreviewCallback previewCallback, void* user_context);

// ライブプレビューの停止
// @param camera_handle カメラハンドル
CSDKError StopPreview(CameraHandle camera_handle);
```

### 1.6 操作コマンド送信

```c
// 操作コマンド（シャッター、AF、録画等）の送信
// @param camera_handle カメラハンドル
// @param command_code 送信するコマンド
// @param command_value コマンドに付随する値 (不要なコマンドの場合は NULL)
CSDKError SendCommandCode(CameraHandle camera_handle, CCommandCode command_code, const CCommandValueUnion* command_value);
```

### 1.7 コンテンツ管理 (Session 1 スコープ対応)

```c
// カメラ内コンテンツの一覧取得
// @param camera_handle カメラハンドル
// @param content_count 取得されたコンテンツの総数 (out)
// @param contents コンテンツ情報の配列の先頭ポインタを受け取る変数へのポインタ (out)
// ※内部でメモリが確保されるため、使用後は必ず ReleaseEnumContents で解放してください。
CSDKError EnumContents(CameraHandle camera_handle, size_t* content_count, CameraContentInfo** contents);

// 列挙で確保したコンテンツ情報のメモリ解放
// @param contents EnumContentsで取得した配列ポインタ
CSDKError ReleaseEnumContents(CameraContentInfo* contents);

// コンテンツのダウンロード（非同期）
// @param camera_handle カメラハンドル
// @param content_info ダウンロード対象のコンテンツ情報
// @param callback ダウンロード進捗および完了を通知するコールバック
// @param user_context コールバックに渡されるユーザー任意のコンテキスト
CSDKError DownloadContent(CameraHandle camera_handle, const CameraContentInfo* content_info, ContentDownloadCallback callback, void* user_context);

// コンテンツの削除
// @param camera_handle カメラハンドル
// @param content_info 削除対象のコンテンツ情報
CSDKError DeleteContent(CameraHandle camera_handle, const CameraContentInfo* content_info);

#ifdef __cplusplus
}
#endif
```

---

## 2. API入出力詳細

### 2.1 SDK初期化 / 解放
* **API名**: `Initialize` / `Release`
* **引数**: なし
* **戻り値**: `CSDKError` (成功時は `CSDKError_Success`)
* **説明**: SDK全体の動作に必要な内部スレッド、ロガー、共通バッファプールなどを準備 / 破棄します。アプリケーション開始時および終了時に必ず1回ずつ呼び出してください。

### 2.2 カメラ検出
* **API名**: `EnumCameras`
* **引数**: 
  * `out size_t* camera_count`: 検出されたカメラの数を書き込むポインタ。
  * `out CameraDeviceInfo** cameras`: 確保された `CameraDeviceInfo` 配列の先頭ポインタのアドレスを書き込むポインタ。
* **戻り値**: `CSDKError`
* **説明**: PCに物理的（USB）、またはネットワーク（IP）経由で接続されている全てのC社製カメラを自動検出し、その基本情報を返します。取得したメモリ領域は、使用後に必ず `ReleaseEnumCameras` に渡して解放する必要があります。

### 2.3 カメラ接続
* **API名**: `ConnectCamera`
* **引数**:
  * `in const CameraDeviceInfo* camera_info`: `EnumCameras` で取得されたもしくはユーザが手動で設定した、接続対象カメラの情報ポインタ。
  * `in ConnectCallback connectCallback`: 接続の完了・成否を非同期で通知するコールバック関数。
  * `in EventCallback eventCallback`: 接続中に発生する一時的なエラーや強制切断などの非同期イベントを監視するコールバック関数。
  * `in void* user_context`: コールバック関数が呼ばれる際に、そのまま渡される呼び出し側定義のコンテキスト（インスタンスポインタ等）。
* **戻り値**: `CSDKError` (接続受付の成否)
* **説明**: 指定したカメラに対する通信スレッドおよび排他制御を立ち上げ、非同期で接続を開始します。実際の接続結果は `ConnectCallback` を介して `CameraHandle` と共に返されます。

### 2.4 カメラ切断
* **API名**: `DisconnectCamera`
* **引数**:
  * `in CameraHandle camera_handle`: 制御対象のカメラハンドル。
* **戻り値**: `CSDKError`
* **説明**: 指定したカメラとの接続を非同期で遮断し、関連する通信スレッドやメモリプール、ハンドルを破棄します。切断の完了イベントは、`ConnectCamera` で登録した `EventCallback` にて通知されます。

### 2.5 カメラパラメータ取得
* **API名**: `GetCameraSetting`
* **引数**: 
  * `in CameraHandle camera_handle`: 制御対象のカメラハンドル。
  * `in/out CProperty* property`: 設定パラメータを取得・格納するための構造体ポインタ。
    * 事前に `property->code` に取得したい設定コードを格納して呼び出します。呼び出しが成功すると、`property->value` に値が格納されます。
* **戻り値**: `CSDKError`

### 2.6 カメラパラメータ適用 (複数一括設定)
* **API名**: `SetCameraSetting`
* **引数**: 
  * `in CameraHandle camera_handle`: カメラデバイスのハンドル
  * `in size_t property_count`: 設定項目の数
  * `in const CProperty* property`: カメラデバイスの設定
* **戻り値**: `CSDKError`
* **説明**: 複数のプロパティ変更要求をカメラに順次送信します。

### 2.7 ライブプレビュー開始 / 終了
* **API名**: `StartPreview` / `StopPreview`
* **引数**:
  * `in CameraHandle camera_handle`: 制御対象のカメラハンドル。
  * `in PreviewCallback previewCallback` (StartPreviewのみ): 新しい画像フレームが届いた際に呼び出されるコールバック関数。
  * `in void* user_context` (StartPreviewのみ): プレビューコールバックに渡される任意のユーザーコンテキスト。
* **戻り値**: `CSDKError`
* **説明**: カメラからのライブプレビューのストリーミングを開始 / 停止します。開始されると、SDK内部の画像受信専用スレッドが起動し、画像フレームを受信し次第 `PreviewCallback` を即座に発火します。

### 2.8 操作コマンド送信
* **API名**: `SendCommandCode`
* **引数**:
  * `in CameraHandle camera_handle`: 制御対象のカメラハンドル。
  * `in CCommandCode command_code`: 送信する操作コマンド（シャッター、AF等）。
  * `in const CCommandValueUnion* command_value`: コマンドに付随する追加値。不要なコマンドの場合は `NULL` を指定します。
* **戻り値**: `CSDKError`

### 2.9 コンテンツ一覧取得
* **API名**: `EnumContents`
* **引数**:
  * `in CameraHandle camera_handle`: 制御対象のカメラハンドル。
  * `out size_t* content_count`: 検出されたコンテンツ（静止画・動画ファイル）の総数を受け取る変数へのポインタ。
  * `out CameraContentInfo** contents`: 確保された `CameraContentInfo` 配列の先頭ポインタのアドレスを受け取るポインタ。
* **戻り値**: `CSDKError`
* **説明**: カメラ内のストレージ（SDカード等）に記録されているメディアファイルの一覧を取得します。取得したメモリ領域は、使用後に必ず `ReleaseEnumContents` に渡して解放する必要があります。

### 2.10 コンテンツダウンロード
* **API名**: `DownloadContent`
* **引数**:
  * `in CameraHandle camera_handle`: 制御対象のカメラハンドル。
  * `in const CameraContentInfo* content_info`: ダウンロード対象のコンテンツ（ファイル名等が含まれる構造体）。
  * `in ContentDownloadCallback callback`: 進捗およびダウンロード完了を非同期で通知するコールバック関数。
  * `in void* user_context`: コールバックに渡される任意のユーザーコンテキスト。
* **戻り値**: `CSDKError` (ダウンロード処理の受付成否)
* **説明**: カメラのストレージから対象ファイルを非同期でダウンロードします。進捗状況や最終完了結果は `ContentDownloadCallback` で通知されます。

### 2.11 コンテンツ削除
* **API名**: `DeleteContent`
* **引数**:
  * `in CameraHandle camera_handle`: 制御対象のカメラハンドル。
  * `in const CameraContentInfo* content_info`: 削除対象のコンテンツ情報のポインタ。
* **戻り値**: `CSDKError`
* **説明**: カメラ内の対象ファイルを削除します。この処理は同期的（呼び出しが返るまで制御が戻らない）に行われます。

---

## 3. 主要なデータ構造

### 3.1 `CSDKError` (エラーコード定義)
上位8ビットでエラーの「カテゴリ」、下位8ビットで「詳細種別」を表します。
```c
enum CSDKError{
// 成功
CSDKError_Success =                       0x0000,

// 一般的なエラーカテゴリ (0x8000~)
CSDKError_General =                       0x8000,
CSDKError_General_InvalidArg =            0x8001,  // 引数が不正
CSDKError_General_NotInitialized =        0x8002,  // SDKが初期化されていない
CSDKError_General_DeviceNotFound =        0x8003,  // 対象のカメラが見つからない
CSDKError_General_InvalidHandle =         0x8005,  // 無効なカメラハンドル
CSDKError_General_NotSupported =          0x8006,  // サポートされていない機能

// 通信系のエラーカテゴリ (0x8100~)
CSDKError_Network =                       0x8100,
CSDKError_Network_ConnectionFailed =      0x8101,  // カメラへの接続に失敗
CSDKError_Network_CommunicationError =    0x8102,  // 通信中のエラー
CSDKError_Network_DeviceError =           0x8103,  // デバイスエラー(全般)
CSDKError_Network_DeviceBusy =            0x8104,  // デバイスエラー(ビジー)
CSDKError_Network_Timeout =               0x8105,  // デバイスエラー(タイムアウト)
CSDKError_Network_NotSupported =          0x8106,  // デバイスエラー(サポート外の操作)

// SDK内部処理のエラーカテゴリ (0x8200~)
CSDKError_Process =                       0x8200,
CSDKError_Process_OutOfMemory =           0x8201,  // メモリ不足
CSDKError_Process_Unexpected =            0x8202,  // 予期せぬ内部エラー
};
```

### 3.2 `InterfaceType` (Enum)
```c
typedef enum {
    INTERFACE_TYPE_USB   = 0,
    INTERFACE_TYPE_ETHER = 1
} InterfaceType;
```

### 3.3 `CameraDeviceInfo` (構造体)
```c
struct CameraDeviceInfo {
    char serial_number[64];      // カメラのユニークなシリアル番号
    char model_name[128];        // モデル名 (例: "C-Model-A")
    InterfaceType interface_type;// 接続インターフェース
    char ip_address[64];         // ネットワーク接続時のIPアドレス (USB時は空)
};
```

### 3.4 `CPropertyCode` (Enum)
```c
typedef enum {
    CPROPERTY_RESOLUTION   = 1,   // 解像度 (例: 1920x1080)
    CPROPERTY_FRAME_RATE   = 2,   // フレームレート (fps)
    CPROPERTY_EXPOSURE_TIME = 3,  // 露光時間 (マイクロ秒)
    CPROPERTY_GAIN          = 4,  // ゲイン値 (dB)
    CPROPERTY_WHITE_BALANCE = 5   // ホワイトバランスモード
} CPropertyCode;
```

### 3.5 `CPropertyValue` (Union)
```c
typedef union {
    int32_t int_val;
    double float_val;
    char str_val[128];
    struct {
        uint32_t width;
        uint32_t height;
    } resolution_val;
} CPropertyValue;
```

### 3.6 `CProperty` (構造体)
```c
struct CProperty {
    CPropertyCode code;     // プロパティ種別
    CPropertyValue value;   // プロパティ値
};
```

### 3.7 `CCommandCode` (Enum)
```c
typedef enum {
    CCOMMAND_SHUTTER       = 1,   // 静止画撮影（シャッターを切る）
    CCOMMAND_START_RECORD  = 2,   // 動画撮影開始
    CCOMMAND_STOP_RECORD   = 3,   // 動画撮影停止
    CCOMMAND_AUTO_FOCUS    = 4    // オートフォーカス実行
} CCommandCode;
```

### 3.8 `CCommandValueUnion` (Union)
```c
union CCommandValueUnion {
    int32_t int_val;
    char str_val[64];
};
```

### 3.9 `CEventType` (Enum) & `CEventData` (構造体)
```c
typedef enum {
    EVENT_TYPE_DEVICE_MISSING   = 1,  // 通信切断（ケーブル抜けなど）
    EVENT_TYPE_DEVICE_CONNECTED = 2,  // カメラ再接続完了
    EVENT_TYPE_STORAGE_FULL     = 3   // ストレージ容量不足警告
} CEventType;

struct CEventData {
    CEventType event_type;      // イベント種別
    CSDKError detail_error;     // エラーが原因の場合のトラブルシューティングコード
};
```

### 3.10 `CLivePreviewData` (構造体) & `CFrameInfo` (構造体)
```c
struct CFrameInfo {
    uint64_t frame_index;       // フレーム番号
    uint64_t timestamp_ns;      // ナノ秒単位のタイムスタンプ
    uint32_t exposure_us;       // このフレーム撮影時の露光時間
    uint32_t gain_db;           // このフレーム撮影時のゲイン
};

struct CLivePreviewData {
    uint32_t width;             // 画像の幅 (ピクセル)
    uint32_t height;            // 画像の高さ (ピクセル)
    uint32_t pixel_format;      // ピクセルフォーマット (RGB, YUV等を示す定数)
    uint8_t* data_buffer;       // 画像データバッファへのポインタ
    size_t data_size;           // バッファサイズ (バイト)
    CFrameInfo frame_info;      // メタデータフレーム情報
};
```

### 3.11 `CameraContentInfo` (構造体)
```c
struct CameraContentInfo {
    char file_name[256];        // ファイル名 (例: "IMG_0001.JPG")
    size_t file_size;           // ファイルサイズ (バイト)
    uint64_t created_time;      // 撮影日時 (UNIXタイムスタンプ)
    int32_t content_type;       // コンテンツ種別 (1: 静止画, 2: 動画)
};
```

---

## 4. 【設計判断】なぜこのAPI単位・粒度にしたのか？

- **CスタイルAPI（Cリンケージ）の採用理由**
  - アプリケーション開発者が使用する多様な言語（C#, Python, Unity等）へのラッパー（Binding）を提供しやすくするためです。C++クラスのABIはコンパイラ間で互換性がないのに対し、`extern "C"` を用いた純粋なC関数APIは、ほぼすべての開発環境と安定してリンクできます。
- **CameraHandle を `uint64_t` とした理由**
  - ユーザーに対し「ポインタであること」を完全に隠蔽し、不透明な識別用IDとして安全に取り扱ってもらうためです。`void*` や `intptr_t` などの表現はポインタの存在を想起させ、不要なポインタ演算や不正な dereference を誘発するリスクがあります。`uint64_t` であれば 32bit/64bit 環境の双方でポインタアドレスを完全に内包でき、型名から "ptr" を排除した完全な不透明ハンドル（Opaque Handle）を実現できます。
- **カメラ接続、ダウンロード、プレビューを非同期（コールバック）にした理由**
  - PTP通信やディスクIO、画像転送などの重たい処理において、呼び出し元のUIスレッドやメインスレッドをブロック（フリーズ）させないためです。API自体は要求の受付（Queueing）を終えたら即座に制御を返し、結果やデータはSDK内部で生成される専用スレッドからコールバックを通じて非同期に通知します。
- **パラメータ設定を汎用プロパティ（CProperty）方式にした理由**
  - 今後カメラのファームウェア更新によって新規設定項目（プロパティ）が増えた場合でも、SDKのAPIシグネチャ（関数シグネチャ）を変更することなく、`CPropertyCode` の追加だけで拡張できるようにするためです。また、個別メソッド方式（`SetExposure`, `SetGain` など）を乱立させるより、一括設定・一括取得の同期がとりやすいメリットがあります。
- **コンテンツ管理APIを追加し、非同期ダウンロードにした理由**
  - Session 1 のユースケース（UC-09/10/11）を満たすために必須の機能です。特に大容量の動画ファイルをダウンロードする際、ネットワークやUSB転送に時間がかかるため、プログレス進捗を通知する `ContentDownloadCallback` を用いた非同期転送設計とし、アプリ側での進捗表示（進捗率%など）を容易にしました。
