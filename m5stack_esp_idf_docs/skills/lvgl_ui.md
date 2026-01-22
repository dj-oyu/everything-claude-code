---
name: lvgl-ui
description: M5Stack Tab5のようなESP32デバイス上で、LVGLグラフィックライブラリを使用してユーザーインターフェース（UI）を構築するためのスキル。
---

# LVGL UI構築スキル

このスキルは、LVGL (Light and Versatile Graphics Library) を使用して、ESP32/ESP32-P4を搭載したM5Stack Tab5のようなデバイス上で、リッチでインタラクティブなユーザーインターフェースを構築する方法をAIエージェントに提供します。

## コアコンセプト

*   **LVGLとは**: 低メモリフットプリントで動作するように設計された、組み込みシステム向けのオープンソースのグラフィックライブラリ。
*   **オブジェクト（ウィジェット）**: UIはオブジェクトの階層で構成されます。基本的なオブジェクト（`lv_obj`）の上に、ボタン（`lv_btn`）、ラベル（`lv_label`）、チャート（`lv_chart`）、スライダー（`lv_slider`）などが構築されます。
*   **親子関係**: 全てのオブジェクトは親オブジェクトを持ちます。親が移動すれば子も移動し、親が削除されれば子も削除されます。`lv_screen_active()`は最上位の親オブジェクト（スクリーン）を返します。
*   **イベント駆動**: UIのインタラクションはイベントシステムに基づいています。オブジェクトにコールバック関数を登録し、`LV_EVENT_CLICKED`や`LV_EVENT_VALUE_CHANGED`などのイベントを処理します。
*   **スタイル**: オブジェクトの見た目（色、フォント、境界線など）はスタイルによって定義されます。

## ESP-IDF LVGL Port ワークフロー

M5Stack Tab5のBSP（ボードサポートパッケージ）はLVGLの初期化を大幅に簡略化しています。

### 1. LVGLとディスプレイの初期化

*   **BSPの利用**: `bsp_display_start()`または`bsp_display_start_with_config()`を呼び出すだけで、LVGLの初期化、MIPI DSIディスプレイドライバの設定、タッチパネル入力デバイスの登録がすべて行われます。
    *   `m5stack_tab5.c`の`bsp_display_start_with_config`関数は、内部で`lvgl_port_init`、`bsp_display_lcd_init`、`bsp_display_indev_init`（または`..._to_st7123`）を呼び出しています。
*   **ディスプレイハンドル**: `bsp_display_start...`関数は、後続の操作で使用できる`lv_display_t*`ハンドルを返します。

### 2. UIオブジェクトの作成

*   **基本**: `lv_obj_t * my_button = lv_btn_create(lv_screen_active());` のように、`lv_..._create(parent)`関数を使用して、現在のスクリーン（または他のコンテナオブジェクト）を親としてウィジェットを作成します。
*   **プロパティの設定**:
    *   サイズと位置: `lv_obj_set_size(my_button, 120, 50);`, `lv_obj_set_align(my_button, LV_ALIGN_CENTER);`
    *   ウィジェット固有のプロパティ: `lv_label_set_text(my_label, "Hello World");`
*   **スタイル**: `lv_obj_add_style()`を使用して、オブジェクトにスタイルを適用します。

### 3. イベントハンドリング

*   **コールバック関数**: `void my_event_handler(lv_event_t * e)` のようなイベントハンドラを定義します。
*   **イベント登録**: `lv_obj_add_event_cb(my_button, my_event_handler, LV_EVENT_CLICKED, NULL);` のように、特定のオブジェクトとイベントタイプにハンドラを関連付けます。
*   **イベントデータ**: イベントハンドラ内で`lv_event_get_code(e)`でイベントの種類を、`lv_event_get_target(e)`でイベントが発生したオブジェクトを取得できます。

### 4. LVGLハンドラタスク

LVGLは、自身のタイマーやイベントを処理するために、定期的に特定の関数を呼び出す必要があります。

*   **専用タスク**: LVGLの処理のために、独立したFreeRTOSタスクを作成するのが一般的です。
*   **メインループ**: このタスクの`while(1)`ループ内で、`lv_timer_handler()`を定期的に（例: 10msごとに）呼び出します。
    ```c
    vTaskDelay(pdMS_TO_TICKS(10));
    ```
*   **ミューテックスによる保護**: 複数のタスクからUIを操作する場合、LVGLのAPI呼び出しはスレッドセーフではありません。必ずミューテックスで保護する必要があります。M5Stack Tab5 BSPは`bsp_display_lock()`と`bsp_display_unlock()`というラッパー関数を提供しています。
    ```c
    if (bsp_display_lock(portMAX_DELAY)) {
        // ... LVGLのUI操作コード ...
        bsp_display_unlock();
    }
    ```
    `lv_timer_handler()`の呼び出しもこのロックの内側で行う必要があります。

## コード例: ボタンとラベルの作成

```c
#include "bsp/m5stack_tab5.h"
#include "lvgl.h"

// グローバルスコープまたはstaticでラベルへのポインタを保持
static lv_obj_t *info_label;

// ボタンクリック時のイベントハンドラ
static void button_event_handler(lv_event_t *e) {
    lv_event_code_t code = lv_event_get_code(e);
    lv_obj_t *btn = lv_event_get_target(e);

    if (code == LV_EVENT_CLICKED) {
        ESP_LOGI("LVGL", "Button Clicked");
        
        // ラベルのテキストを更新
        static uint32_t counter = 0;
        counter++;
        if (bsp_display_lock(portMAX_DELAY)) {
            lv_label_set_text_fmt(info_label, "Button clicked: %d times", counter);
            bsp_display_unlock();
        }
    }
}

void create_simple_ui(void) {
    // ディスプレイの初期化（アプリケーションの開始時に一度だけ呼び出す）
    lv_display_t *disp = bsp_display_start();

    // 画面の背景色を変更
    lv_obj_set_style_bg_color(lv_screen_active(), lv_color_hex(0x282c34), LV_PART_MAIN);

    // --- ボタンの作成 ---
    lv_obj_t *my_button = lv_btn_create(lv_screen_active());
    lv_obj_set_pos(my_button, 100, 100);
    lv_obj_set_size(my_button, 200, 80);
    lv_obj_add_event_cb(my_button, button_event_handler, LV_EVENT_ALL, NULL);

    lv_obj_t *btn_label = lv_label_create(my_button);
    lv_label_set_text(btn_label, "Click Me!");
    lv_obj_center(btn_label);

    // --- ラベルの作成 ---
    info_label = lv_label_create(lv_screen_active());
    lv_label_set_text(info_label, "Press the button.");
    lv_obj_set_style_text_font(info_label, &lv_font_montserrat_24, 0);
    lv_obj_set_style_text_color(info_label, lv_color_white(), 0);
    lv_obj_align(info_label, LV_ALIGN_CENTER, 0, 100);
}
```

## M5Stack Tab5特有の事項

*   **初期化の簡略化**: `bsp_display_start()`を呼び出すだけで、LVGL、MIPI DSIディスプレイ、タッチパネルの複雑な初期化が完了します。
*   **ティアリング防止**: M5Stack Tab5のBSPでは、`lvgl_port_add_disp_dsi`の`dpi_cfg`フラグで`.avoid_tearing = true`が設定されており、ティアリングの少ない滑らかな描画がデフォルトで期待できます。
*   **ダブルバッファリング**: 同様に、`.double_buffer = true`がデフォルトで有効になっており、パフォーマンス向上に寄与しています。

## デバッグのヒント

*   **LVGLのログ**: `menuconfig`の`Component config -> LVGL -> LVGL general configuration`でログレベルを`Debug`や`Trace`に設定すると、詳細なログが出力されます。
*   **シンプルなUIから始める**: 複雑な画面を作る前に、まず一つのラベルやボタンが正しく表示・動作するかを確認します。
*   **`lv_timer_handler()`の呼び出し確認**: この関数が定期的に呼び出されていないと、UIが一切更新されません（アニメーション、再描画、入力処理など）。タスクがブロックされていないか確認してください。
*   **スタックサイズの確保**: LVGLを使用するタスクには、十分なスタックサイズ（例: 8192バイト以上）を割り当ててください。スタック不足は予期せぬクラッシュの原因になります。
*   **ミューテックスの確認**: UIがフリーズしたり、表示が崩れたりする場合、複数のタスクからUIを操作する際に`bsp_display_lock()` / `bsp_display_unlock()`で保護し忘れていないか確認してください。
