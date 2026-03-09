# AGENTS.md - ESP32-C5 SmartButton

## Project Overview

IoT dual smart button for **ESP32-C5** (RISC-V, WiFi 6) built with **ESP-IDF v6.1** in C.
Two physical buttons trigger configurable HTTP requests or MQTT publishes.
Features: deep sleep with EXT1 wakeup, captive portal, OTA updates, WS2812 RGB LED status, web-based configuration panel with Basic Auth.

Project name: `smartbutton_c5`

## Build & Flash

Requires ESP-IDF v6.1 with the `esp32c5` target. The target is set in `sdkconfig.defaults`.

```bash
idf.py set-target esp32c5   # Only needed once
idf.py build
idf.py flash monitor        # Flash and open serial monitor
idf.py monitor              # Serial monitor only (115200 baud)
```

Custom partition table in `partitions.csv` (4MB flash: dual OTA partitions of 1700K each, 24K NVS, 300K SPIFFS storage).

## Code Structure

```
main/main.c                     # Entry point (app_main): boot detection, init, superloop, deep sleep logic
components/
  app_core/                     # State machine (system_state_t enum), event group, state change callback
  app_nvs/                      # NVS persistence: WiFi, MQTT, button configs, admin settings
  app_wifi/                     # WiFi STA/AP/APSTA modes, scan, fallback to AP on STA failure
  app_web/                      # HTTP server, REST API endpoints, Basic Auth, OTA handler
    html_ui.h                   # Full web UI as a C raw string literal (R"rawliteral(...)")
  app_http/                     # Outbound HTTP client (GET/POST) for button actions
  app_mqtt/                     # Oneshot MQTT publish (connect → publish → disconnect)
  app_buttons/                  # Button polling task with debounce, cooldown, factory reset combo
  app_led/                      # WS2812 LED control via RMT, blink patterns per state
  app_dns/                      # Captive portal DNS server (responds all queries with 192.168.4.1)
```

Each component follows the pattern: `app_xxx.c` + `include/app_xxx.h` + `CMakeLists.txt`.

## Hardware Pins

| Function | GPIO | Notes |
|----------|------|-------|
| BTN1     | 4    | Active-low, internal pull-up |
| BTN2     | 5    | Active-low, internal pull-up |
| WS2812 LED | 27 | Single addressable RGB LED via RMT |

## Key Patterns

### State Machine & Event Groups
- Global `app_event_group` (FreeRTOS EventGroup) carries system events: `EVENT_WIFI_CONNECTED`, `EVENT_WIFI_LOST`, `EVENT_BUTTON_TRIGGER`, `EVENT_FACTORY_RESET`, `EVENT_HTTP_DONE`.
- `system_state_t` enum in `app_core.h` defines states: `STATE_INIT`, `STATE_NO_CONFIG`, `STATE_AP_MODE`, `STATE_CONNECTING`, `STATE_NORMAL`, `STATE_HTTP_REQ`, `STATE_RESET_WARNING`, `STATE_FACTORY_RESET`, `STATE_ERROR`.
- State transitions go through `app_set_state()` which notifies the LED callback synchronously.

### LED Task (xTaskNotify)
- The LED task uses `xTaskNotifyWait()` as an interruptible delay, allowing instant state transitions.
- `override_led` flag lets `app_led_signal_success()` / `app_led_signal_error()` temporarily take over the LED with blocking visual sequences.
- Always call `notify_led_task()` after changing LED parameters to wake the task immediately.

### Button Polling Task
- Polling at 50ms (`POLL_RATE_MS`) — no interrupts. Debounce is configurable via NVS (default 200ms).
- Single button release triggers action only when `STATE_NORMAL` and cooldown has elapsed.
- Both buttons held simultaneously for `reset_time_ms` (default 8s) triggers factory reset. Warning state starts 3s before reset.

### Oneshot MQTT Pattern
- `app_mqtt_publish_oneshot()` spawns a task that: creates client → connects → publishes (QoS 1) → waits PUBACK → disconnects → destroys client.
- Uses a local `EventGroupHandle_t` within the task to synchronize the async MQTT events.
- The `button_config_t` is `malloc`'d and copied for the task, then `free`'d inside it.
- Reuses `EVENT_HTTP_DONE` bit to signal completion to the main superloop (regardless of HTTP or MQTT action type).

### HTTP Action Tasks
- `app_http_trigger()` creates a one-shot FreeRTOS task (`http_execute_task`) that performs the request and self-deletes.
- Task stack is 12288 bytes (TLS overhead for HTTPS via `esp_crt_bundle`).
- Sets `EVENT_HTTP_DONE` on completion so the main loop can trigger deep sleep.

### Deep Sleep Flow
1. `app_main` configures GPIO pull-ups BEFORE releasing `gpio_hold` (prevents floating pins).
2. On wakeup: detects which button via `esp_sleep_get_ext1_wakeup_status()`, falls back to physical GPIO read.
3. Deferred action: waits for WiFi to connect (`STATE_NORMAL`), then calls `app_buttons_simulate_press()`.
4. Sleep entry: turns off LED → 500ms delay (WS2812 data flush) → `esp_sleep_enable_ext1_wakeup(ANY_LOW)` → `gpio_hold_en()` → `esp_deep_sleep_start()`.

## NVS Namespaces & Key Names

| Namespace    | Keys |
|-------------|------|
| `wifi_conf`  | `ssid`, `pass` |
| `mqtt_conf`  | `host`, `port`, `user`, `pass`, `clid`, `en` |
| `btn_1`, `btn_2` | `atype`, `url` (legacy name for target), `method`, `payload`, `timeout`, `cooldown`, `nocache` |
| `admin_conf` | `user`, `pass`, `reset_ms`, `ap_ssid`, `ap_pass`, `pure_cli`, `deep_slp`, `sta_retr`, `ap_chan`, `wk_tout`, `cfg_awake`, `debounce` |

**NVS key names are max 15 chars.** Abbreviated keys are intentional and must remain stable for backward compatibility with deployed devices.

## REST API Endpoints

All endpoints except `/` and captive portal redirects require Basic Auth.

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/` | Serve web UI (no auth) |
| GET | `/api/verify` | Validate credentials |
| GET | `/api/scan` | WiFi network scan |
| POST | `/api/wifi` | Save WiFi config (triggers restart) |
| GET/POST | `/api/mqtt` | Get/save MQTT broker config |
| GET/POST | `/api/btn?id=N` | Get/save button N config |
| POST | `/api/test` | Test a button action synchronously |
| GET | `/api/netinfo` | Network info (IP, MAC, RSSI) |
| GET/POST | `/api/admin` | Get/save admin settings (triggers restart) |
| POST | `/api/factory_reset` | Erase NVS and restart |
| POST | `/api/ota` | Upload firmware binary |

## Web UI

The entire HTML/CSS/JS web panel lives in `components/app_web/html_ui.h` as a single C raw string literal:
```c
const char index_html[] = R"rawliteral(<!DOCTYPE html>...)rawliteral";
```
- It is a **single-page application** — all HTML, CSS, and JavaScript in one string.
- Edit this file directly to change the UI. No build step or bundler is needed.
- Be careful with the `R"rawliteral(` / `)rawliteral"` delimiters — the content cannot contain the exact sequence `)rawliteral"`.
- The UI communicates with the device via `fetch()` calls to the `/api/*` endpoints with Basic Auth headers.

## Common Pitfalls

1. **NVS key length**: NVS keys must be ≤15 characters. Using longer keys silently fails or corrupts data.
2. **Button target field is stored as `url` key**: The NVS key is `"url"` (legacy name) but the struct field is `target`. Do not rename the NVS key — it would break existing deployed configs.
3. **Deep sleep GPIO hold order**: Pull-ups MUST be configured before calling `gpio_hold_dis()`. Reversing this causes floating reads and phantom wakeups.
4. **500ms delay before deep sleep**: Required for the WS2812 RMT driver to finish transmitting the "LED off" command. Removing it causes the LED to stay on during sleep.
5. **RISC-V and nested functions**: ESP32-C5 is RISC-V — nested/local function pointers (GCC extension) cause crashes. Use static functions with `handler_args` context instead (see `mqtt_sync_handler`).
6. **HTTP task stack size**: The HTTP client task needs 12288 bytes due to TLS (mbedtls). Reducing it causes stack overflow crashes during HTTPS requests.
7. **`EVENT_HTTP_DONE` is shared**: Both HTTP and MQTT action completion set this same event bit. The main loop does not distinguish between action types — it only cares that "the action finished."
8. **`g_wakeup_btn` extern**: `app_buttons.c` references `extern volatile int g_wakeup_btn` — this must be defined somewhere accessible (used to suppress duplicate trigger on wakeup).
9. **Admin/WiFi POST endpoints restart the device**: `esp_restart()` is called after a delay. The HTTP response is sent before the restart, but any follow-up requests will fail.
10. **AP password < 8 chars = open network**: WPA2 requires ≥8 character passwords. Shorter passwords result in an open AP (by design).

## Conventions

- Language: C (C11). No C++ in this project.
- Comments and log messages are in Spanish.
- All components use `static const char *TAG = "XXX"` for ESP logging.
- Boolean config values are stored in NVS as `uint8_t` (0/1).
- Integer config values use `int32_t` in NVS (`nvs_set_i32` / `nvs_get_i32`).
- JSON handling uses the bundled `cJSON` library (provided by ESP-IDF).
- All FreeRTOS tasks run at priority 5.
- Header guards use `#pragma once`.
