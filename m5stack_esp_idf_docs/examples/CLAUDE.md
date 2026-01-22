# プロジェクトCLAUDE.md (M5Stack Tab5 ESP-IDF C開発向け)

これは、M5Stack Tab5、ESP-IDF、C言語開発プロジェクト向けのプロジェクトレベルCLAUDE.mdファイルです。プロジェクトのルートに配置してください。

## プロジェクト概要

M5Stack Tab5 (ESP32-P4 & ESP32-C6) をESP-IDFとC言語で開発するための組み込みソフトウェアプロジェクトです。高性能なディスプレイ、マルチメディア機能、豊富なペリフェラルを活用し、リアルタイム性とリソース効率を重視したアプリケーションを構築します。

## クリティカルなルール

### 1. コード構成

-   機能/ドメインごとに整理し、ファイルの結合度を低く保ちます。
-   一つのファイルは通常200-400行、最大でも800行程度に収めます。
-   BSP (`m5stack_tab5.c`など) からの抽象化を活用し、ボード固有のコードは最小限に留めます。

### 2. コーディング規約 (C言語)

-   `rules/coding_style_c.md`に準拠します。
-   固定幅整数型 (`uint8_t`, `int32_t`など) を常に使用します。
-   変数宣言時に必ず初期化し、`const`を積極的に活用します。
-   `static inline`関数を複雑なマクロよりも優先します。

### 3. テスト

-   組み込み環境のTDD (`rules/rtos_patterns.md`のガイドラインに従います)。
-   主要なモジュールとFreeRTOSタスクは単体テストでカバーします。
-   ハードウェアインタラクションは、モックまたはループバックテストで検証します。
-   重要なデバイスドライバやシステム機能には80%以上のカバレッジを目指します。

### 4. セキュリティ

-   `rules/memory_management.md`を参照し、メモリ管理の脆弱性がないことを確認します。
-   ファームウェアの整合性 (`Secure Boot`, `Flash Encryption`) の利用を検討します。
-   デバッグポートは本番環境で無効化します。

## ファイル構造

```
main/                   # メインアプリケーションコンポーネント
├── main.c              # アプリケーションエントリポイント
├── Kconfig             # menuconfigの設定
├── CMakeLists.txt      # コンポーネントのビルド設定
├── components/         # カスタムコンポーネント（ドライバ、サービスなど）
│   ├── display_task/   # ディスプレイ処理タスク
│   ├── sensor_driver/  # センサー駆動
│   └── ui_manager/     # LVGL UI管理
└── partitions.csv      # パーティションテーブルの定義
```

## 主要なパターン

### エラーハンドリング (esp_err_t)

*   `rules/coding_style_c.md`に詳述されているように、`esp_err_t`を戻り値とし、`ESP_ERROR_CHECK()`を適切に使用します。
    ```c
    esp_err_t my_driver_init(void) {
        esp_err_t ret = some_peripheral_init();
        if (ret != ESP_OK) {
            ESP_LOGE(TAG, "Peripheral init failed: %s", esp_err_to_name(ret));
            return ret;
        }
        ESP_ERROR_CHECK(another_critical_init()); // 致命的な場合はここで停止
        return ESP_OK;
    }
    ```

### メモリ管理 (heap_caps_malloc)

*   `rules/memory_management.md`に従い、`heap_caps_malloc()`で適切なメモリタイプを選択し、`NULL`チェックと`free()`を徹底します。
    ```c
    char *dma_buffer = heap_caps_malloc(1024, MALLOC_CAP_DMA);
    if (dma_buffer == NULL) {
        ESP_LOGE(TAG, "Failed to allocate DMA buffer");
        return ESP_ERR_NO_MEM;
    }
    // ... 使用 ...
    free(dma_buffer);
    ```

## 環境変数 (ESP-IDF)

```bash
# ESP-IDFのパス（通常はセットアップスクリプトで設定）
IDF_PATH=~/esp/esp-idf

# ESP-IDFターゲット（例：esp32p4）
IDF_TARGET=esp32p4
```

## 利用可能なコマンド

-   `/build-flasher build` - プロジェクトをビルドし、メモリ使用量を報告します。
-   `/build-flasher flash` - ビルドされたファームウェアをデバイスに書き込みます。
-   `/build-flasher monitor` - デバイスのシリアル出力を監視します。
-   `/build-flasher all` - ビルド、書き込み、監視を連続して実行します。
-   `/memory-optimizer optimize` - メモリ使用状況を分析し、最適化を提案します。

## Gitワークフロー

-   Conventional Commits: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:` を使用します。
-   メインブランチには直接コミットせず、プルリクエスト経由でマージします。
-   プルリクエストは`code-reviewer`エージェントによるレビューを必要とし、全てのテストがパスしている必要があります。
