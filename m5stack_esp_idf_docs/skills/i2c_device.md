---
name: i2c-device
description: ESP-IDFのI2Cマスタードライバを使用して、センサーやIOエキスパンダーなどのI2Cペリフェラルデバイスと通信するためのスキル。
---

# I2Cデバイス通信スキル

このスキルは、ESP-IDFのI2Cマスタードライバを使用して、I2C（Inter-Integrated Circuit）プロトコルで通信するペリフェラルデバイス（センサー、EEPROM、IOエキスパンダー、電源管理ICなど）を制御する方法をAIエージェントに提供します。

## コアコンセプト

*   **I2C**: 2本のワイヤ、SDA（Serial Data）とSCL（Serial Clock）を使用して、複数のデバイスと通信するための同期式シリアル通信プロトコル。
*   **主要な信号線**:
    *   `SCL` (Serial Clock): マスターが生成するクロック信号。
    *   `SDA` (Serial Data): マスターとスレーブ間の双方向データライン。
*   **アドレス**: 各スレーブデバイスは、バス上で一意の7ビット（または10ビット）アドレスを持つ。
*   **通信**: マスターがSTARTコンディションで通信を開始し、スレーブアドレスを送信。データ送受信後、STOPコンディションで通信を終了する。各バイトの送信後、受信側はACK（Acknowledge）またはNACK（Not Acknowledge）を返す。
*   **プルアップ抵抗**: SDAとSCLラインには、バスがアイドル状態のときに信号線をHighに保つためのプルアップ抵抗が必須です。

## ESP-IDF I2Cマスタードライバ ワークフロー

### 1. バスの初期化

*   **設定**: `i2c_master_bus_config_t`構造体に、`sda_io_num`, `scl_io_num`, `i2c_port`などを設定します。内蔵プルアップを有効にすることも可能です (`.flags.enable_internal_pullup = true`)。
*   **初期化**: `i2c_new_master_bus()`を呼び出して、I2Cマスターバスのハンドル（`i2c_master_bus_handle_t`）を取得します。
*   **M5Stack Tab5例**: `bsp_i2c_init`関数がこの役割を担っています。

### 2. デバイスの追加

*   **設定**: `i2c_device_config_t`構造体に、`device_address`と`scl_speed_hz`を設定します。
*   **追加**: `i2c_master_bus_add_device()`を呼び出し、デバイスハンドル（`i2c_master_dev_handle_t`）を取得します。このハンドルが特定のデバイスとの通信に使用されます。
*   **M5Stack Tab5例**: `bsp_io_expander_pi4ioe_init`関数内で、`I2C_DEV_ADDR_PI4IOE1` (0x43) と `I2C_DEV_ADDR_PI4IOE2` (0x44) のアドレスを持つ2つのI/Oエキスパンダーがバスに追加されています。

### 3. データ送受信

デバイスハンドルを使用して、デバイスと通信します。

*   **書き込み**: `i2c_master_transmit()`: マスターからスレーブにデータを送信します。
*   **読み取り**: `i2c_master_receive()`: スレーブからデータを読み取ります。
*   **書き込み & 読み取り**: `i2c_master_transmit_receive()`: レジスタアドレスを書き込んだ後にデータを読み取る、というセンサーデバイスで一般的な操作を一度に行います。

### 4. クリーンアップ

*   `i2c_master_bus_remove_device()`: デバイスハンドルをバスから削除します。
*   `i2c_del_master_bus()`: I2Cバスのハンドルを解放します。

## コード例: I/Oエキスパンダーへの書き込みと読み取り

```c
#include "driver/i2c.h"

// M5Stack Tab5のbsp_io_expander_pi4ioe_init関数を参考にした例
#define IO_EXPANDER_ADDR 0x43
#define REG_IO_DIR       0x03
#define REG_OUT_SET      0x05

esp_err_t configure_io_expander(i2c_master_bus_handle_t bus_handle) {
    esp_err_t ret;
    i2c_master_dev_handle_t dev_handle;

    // 1. & 2. バスを初期化し、デバイスを追加 (この例ではバスは初期化済みと仮定)
    i2c_device_config_t dev_cfg = {
        .dev_addr_length = I2C_ADDR_BIT_LEN_7,
        .device_address = IO_EXPANDER_ADDR,
        .scl_speed_hz = 400000, // 400 kHz
    };
    ret = i2c_master_bus_add_device(bus_handle, &dev_cfg, &dev_handle);
    if (ret != ESP_OK) return ret;

    // 3. データ送信 (トランザクション)
    uint8_t write_buf[2];

    // IO方向を設定 (例: ポート0を入力、他を出力に)
    write_buf[0] = REG_IO_DIR; // レジスタアドレス
    write_buf[1] = 0xFE;       // データ (Bit0=0 -> Input, 他=1 -> Output)
    ret = i2c_master_transmit(dev_handle, write_buf, sizeof(write_buf), -1); // -1 for blocking
    if (ret != ESP_OK) {
        ESP_LOGE("I2C", "Failed to set IO direction");
        goto cleanup;
    }

    // 出力ポートの状態を設定 (例: 全てHighに)
    write_buf[0] = REG_OUT_SET;
    write_buf[1] = 0xFF;
    ret = i2c_master_transmit(dev_handle, write_buf, sizeof(write_buf), -1);
    if (ret != ESP_OK) {
        ESP_LOGE("I2C", "Failed to set output port");
        goto cleanup;
    }

cleanup:
    // 4. クリーンアップ
    i2c_master_bus_remove_device(dev_handle);
    return ret;
}
```

## 注意すべきパラメータと落とし穴

*   **デバイスアドレス**: 7ビットアドレスは、データシートで確認する必要があります。よくある間違いは8ビットアドレス（R/Wビットを含む）を誤って使用することです。
*   **プルアップ抵抗**: I2Cはオープンドレイン方式のため、SDAとSCLの両ラインにプルアップ抵抗が必要です（通常2.2kΩ〜10kΩ）。ESP-IDFの`i2c_master_bus_config_t`で`.flags.enable_internal_pullup = true`を設定できますが、特に高速モードやバスが長い場合は外部プルアップ抵抗が推奨されます。
*   **クロック周波数 (`scl_speed_hz`)**: バスに接続されている全てのデバイスがサポートする最も遅い速度に合わせる必要があります。一般的には100kHz (Standard Mode) または 400kHz (Fast Mode) です。
*   **ACK/NACK**: `i2c_master_transmit`などの関数は、スレーブからのACKがなかった場合（デバイスが応答しない、アドレスが違うなど）にエラーを返します。戻り値のチェックは必須です。

## デバッグのヒント

*   **I2Cスキャナ**: 通信ができない場合、まず最初にI2Cスキャナを実行して、期待するアドレスのデバイスがバス上で応答するかを確認します。`m5stack_tab5.c`内の`bsp_i2c_scan()`が非常に良い例です。
*   **配線の確認**: SDAとSCLの配線が正しいか、GNDが共通になっているかを確認します。
*   **プルアップ抵抗の確認**: 適切な値のプルアップ抵抗が接続されているか確認します。抵抗値が大きすぎると信号の立ち上がりが遅くなり、小さすぎるとドライブ電流が過大になります。
*   **ロジックアナライザ**:
    *   START/STOPコンディションが正しく生成されているか。
    *   マスターが送信したアドレスが正しいか。
    *   スレーブがACKを返しているか。
    *   データが正しく送受信されているか。
    *   SCLとSDAの波形がなまっていないか（プルアップ抵抗が適切か）。
*   **単一デバイスでのテスト**: バスに複数のデバイスがある場合は、問題のデバイスだけを接続してテストし、アドレスの競合や他のデバイスからの干渉がないか切り分けます。
