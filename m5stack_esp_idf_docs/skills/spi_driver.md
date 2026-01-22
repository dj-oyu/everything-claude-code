---
name: spi-driver
description: ESP-IDFのSPIマスタードライバを使用して、SPIペリフェラルデバイスと通信するためのスキル。バスの初期化、デバイスの追加、トランザクションの実行に関する具体的な手順とパターンを提供。
---

# SPIドライバスキル

このスキルは、ESP-IDFのSPIマスタードライバを使用して、センサー、ディスプレイ、SDカードなどのSPIペリフェラルデバイスと通信するための実装方法をAIエージェントに提供します。

## コアコンセプト

*   **SPI (Serial Peripheral Interface)**: マイクロコントローラとペリフェラルデバイス間の高速な同期式シリアル通信プロトコル。
*   **主要な信号線**:
    *   `SCLK` (Serial Clock): マスターが生成するクロック信号。
    *   `MOSI` (Master Out, Slave In): マスターからスレーブへのデータライン。
    *   `MISO` (Master In, Slave Out): スレーブからマスターへのデータライン。
    *   `CS` (Chip Select): 特定のスレーブデバイスを選択するための信号線。アクティブローが一般的。
*   **Full-duplex**: 送信と受信を同時に行うことができます。

## ESP-IDF SPIマスタードライバ ワークフロー

SPIデバイスとの通信は、以下の4つのステップで行います。

### 1. バス (Host) の初期化

まず、SPIバス自体を初期化します。バスは複数のデバイスで共有できます。

*   **設定**: `spi_bus_config_t`構造体に、`mosi_io_num`, `miso_io_num`, `sclk_io_num`などの物理的なGPIOピン番号を設定します。
*   **初期化**: `spi_bus_initialize()`を呼び出して、設定を適用しバスを初期化します。

### 2. デバイスの追加

次に、初期化したバスに通信したい個別のデバイスを追加します。

*   **設定**: `spi_device_interface_config_t`構造体に、デバイス固有の設定を行います。
    *   `command_bits`, `address_bits`: コマンドやアドレスフェーズの長さを設定。
    *   `clock_speed_hz`: SPIクロック周波数（デバイスのデータシートを確認）。
    *   `mode`: SPIモード (0-3)。クロックの極性(CPOL)と位相(CPHA)の組み合わせ。
    *   `spics_io_num`: このデバイス用のチップセレクト(CS)ピン番号。
    *   `queue_size`: （割り込みモードの場合）キューイングできるトランザクション数。
*   **追加**: `spi_bus_add_device()`を呼び出し、`spi_device_handle_t`（デバイスハンドル）を取得します。以降の通信はこのハンドルを使用します。

### 3. トランザクションの実行

デバイスハンドルを使用して、実際のデータ送受信（トランザクション）を実行します。

*   **設定**: `spi_transaction_t`構造体に、送受信するデータを設定します。
    *   `tx_buffer`: 送信するデータへのポインタ。
    *   `rx_buffer`: 受信したデータを格納するバッファへのポインタ。
    *   `length`: 送信するビット長。
    *   `rxlength`: 受信するビット長（`length`と同じことが多い）。
*   **実行**:
    *   **ポーリング（単純・ブロッキング）**: `spi_device_polling_transmit()`を使用します。完了するまでブロックされます。
    *   **割り込み（非同期）**: `spi_device_queue_trans()`でトランザクションをキューに入れ、後で`spi_device_get_trans_result()`で結果を取得します。RTOSタスクと組み合わせるのに適しています。

### 4. クリーンアップ

通信が不要になったら、リソースを解放します。

*   `spi_bus_remove_device()`: デバイスハンドルをバスから削除します。
*   `spi_bus_free()`: バスを解放します。

## コード例: SPIデバイスとのデータ送受信

```c
#include "driver/spi_master.h"

// SPIピン設定の例
#define SPI_HOST    SPI2_HOST
#define PIN_NUM_MISO 13
#define PIN_NUM_MOSI 11
#define PIN_NUM_SCLK 12
#define PIN_NUM_CS   10

esp_err_t spi_communicate_with_sensor(void) {
    esp_err_t ret;
    spi_device_handle_t spi_handle;

    // 1. バスの初期化
    spi_bus_config_t buscfg = {
        .miso_io_num = PIN_NUM_MISO,
        .mosi_io_num = PIN_NUM_MOSI,
        .sclk_io_num = PIN_NUM_SCLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = 32,
    };
    ret = spi_bus_initialize(SPI_HOST, &buscfg, SPI_DMA_CH_AUTO);
    ESP_ERROR_CHECK(ret);

    // 2. デバイスの追加
    spi_device_interface_config_t devcfg = {
        .clock_speed_hz = 10 * 1000 * 1000, // 10 MHz
        .mode = 0,                         // SPI mode 0
        .spics_io_num = PIN_NUM_CS,        // CSピン
        .queue_size = 7,                   // 7個のトランザクションをキューイング
    };
    ret = spi_bus_add_device(SPI_HOST, &devcfg, &spi_handle);
    ESP_ERROR_CHECK(ret);

    // 3. トランザクションの実行
    uint8_t tx_data[2] = {0x9F, 0x00}; // JEDEC IDを読み出すコマンドの例
    uint8_t rx_data[4] = {0};

    spi_transaction_t t;
    memset(&t, 0, sizeof(t));
    t.length = 8 * sizeof(tx_data); // 長さはビット単位
    t.tx_buffer = tx_data;
    t.rx_buffer = rx_data;
    t.rxlength = 8 * sizeof(rx_data);

    // ポーリング方式でトランザクションを実行
    ret = spi_device_polling_transmit(spi_handle, &t);
    assert(ret == ESP_OK);

    ESP_LOGI("SPI", "Received data: %02X %02X %02X %02X", rx_data[0], rx_data[1], rx_data[2], rx_data[3]);

    // 4. クリーンアップ
    ret = spi_bus_remove_device(spi_handle);
    ESP_ERROR_CHECK(ret);
    ret = spi_bus_free(SPI_HOST);
    ESP_ERROR_CHECK(ret);

    return ret;
}
```

## 注意すべきパラメータと落とし穴

*   **`clock_speed_hz`**: デバイスのデータシートで最大クロック周波数を確認してください。高すぎるとデータが化けます。
*   **`mode`**: SPIモード(0, 1, 2, 3)はデバイスの仕様と正確に一致させる必要があります。クロックの極性(CPOL)と位相(CPHA)が違うと通信できません。
*   **`queue_size`**: 割り込みベースのトランザクションを使用する場合に重要です。同時にキューイングしたいトランザクションの数を指定します。
*   **DMAの使用**: `spi_bus_initialize`の3番目の引数でDMAチャネルを指定します。大きなデータを転送する場合はDMAが必須です。DMAを使用するバッファは`MALLOC_CAP_DMA`フラグ付きで確保する必要があります。
*   **`max_transfer_sz`**: 一度のトランザクションで転送できる最大バイト数。DMAを使用しない場合、この値は小さく（例: 64バイト）なります。大きなデータを送るには複数回に分けるか、DMAを有効にする必要があります。

## デバッグのヒント

*   **ピン割り当ての再確認**: 回路図と`spi_bus_config_t`の設定が完全に一致しているか確認してください。特にMOSIとMISOの接続ミスはよくあります。
*   **低速から始める**: 最初は`clock_speed_hz`を非常に低い値（例: 100kHz）に設定して通信を試し、成功したら徐々に上げていきます。
*   **ロジックアナライザの活用**: SPI通信のデバッグに最も強力なツールです。SCLK, MOSI, MISO, CSの各信号線をプローブし、以下の点を確認します。
    *   CSが正しくアクティブローになっているか。
    *   クロック信号が期待通りの周波数で出ているか。
    *   MOSIラインに正しいコマンドやデータが出力されているか。
    *   MISOラインから応答があるか。
*   **CSピンの手動制御**: `spics_io_num`を`-1`に設定し、`gpio_set_level`でCSピンを手動制御することで、トランザクション間のタイミングを細かく調整できます。
