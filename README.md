# Universal Keyboard Driver Framework

[English](#english) | [中文](#中文)

---

## English

### Overview

A production-ready, universal keyboard driver framework for embedded systems. Supports multiple input methods including independent GPIO keys, matrix keyboards, and custom scan interfaces. Features comprehensive event detection (click, double-click, long-press, repeat) with built-in debouncing and efficient memory management.

### Key Features

- **🎯 Universal Hardware Abstraction**
  - Independent GPIO keys
  - Matrix keyboard (row-column scanning)
  - Custom scan interface (I2C/SPI chips, etc.)

- **⚡ Rich Event Detection**
  - Press / Release
  - Click / Double-click
  - Long-press / Long-press release
  - Auto-repeat

- **🛡️ Robust Design**
  - Software debouncing
  - Optional lock/unlock hooks for key registration
  - Multiple keys simultaneous detection
  - Matrix raw scan support (software anti-ghosting is not built in)

- **💾 Efficient Memory Management**
  - Custom memory pool (no fragmentation)
  - Configurable pool size
  - Static allocation friendly

- **🔧 Easy Integration**
  - Clean API design
  - Minimal dependencies
  - Comprehensive error handling

### Architecture

```
┌─────────────────────────────────────────┐
│         Application Layer               │
│  (Event callbacks, key combinations)    │
└─────────────────┬───────────────────────┘
                  │ Events
┌─────────────────▼───────────────────────┐
│      Keyboard Driver Framework          │
│  • Event detection (click/long/repeat)  │
│  • Debouncing algorithm                 │
│  • State management                     │
└─────────────────┬───────────────────────┘
                  │ Hardware abstraction
┌─────────────────▼───────────────────────┐
│       Hardware Backend (User)           │
│  • GPIO: read_pin()                     │
│  • Matrix: select_row/read_col()        │
│  • Custom: scan_snapshot()              │
└─────────────────────────────────────────┘
```

### Quick Start

#### 1. GPIO Mode Example

```c
#include "keyboard_driver.h"

// Hardware layer: GPIO read function
uint8_t my_read_pin(uint8_t pin) {
    return gpio_read(pin);  // Your GPIO driver
}

// Event callback
void on_key_event(const char *keyname, uint16_t key_id,
                  kb_event_t evt, void *user) {
    switch (evt) {
        case KB_EVT_CLICK:
            printf("Key %s clicked\n", keyname);
            break;
        case KB_EVT_LONGPRESS:
            printf("Key %s long-pressed\n", keyname);
            break;
        // ... handle other events
    }
}

int main(void) {
    keyboard_control_t kb_ctl;

    // 1. Setup hardware operations
    keyboard_ops_t ops = {
        .read_pin = my_read_pin,
        .get_tick_ms = NULL,  // Optional
        .lock = NULL,         // Optional
        .unlock = NULL
    };

    // 2. Setup callback
    keyboard_cb_t cb = {
        .on_event = on_key_event,
        .user = NULL
    };

    // 3. Initialize
    keyboard_init(&kb_ctl, &ops, &cb);

    // 4. Register keys
    keyboard_register_gpio(0, "KEY_A", 0x01, &kb_ctl);
    keyboard_register_gpio(1, "KEY_B", 0x02, &kb_ctl);

    // 5. Poll in main loop (every 10ms)
    while (1) {
        keyboard_poll(&kb_ctl, 10);
        delay_ms(10);
    }
}
```

#### 2. Matrix Keyboard Example

```c
#include "keyboard_driver.h"

static uint8_t row_pins[4] = {0, 1, 2, 3};  // replace with your row pins
static uint8_t col_pins[4] = {4, 5, 6, 7};  // replace with your col pins

static uint8_t map_row(uint8_t logical_row) {
#if KB_MATRIX_ROW_REVERSE
    return (uint8_t)((4u - 1u) - logical_row);
#else
    return logical_row;
#endif
}

static uint8_t map_col(uint8_t logical_col) {
#if KB_MATRIX_COL_REVERSE
    return (uint8_t)((4u - 1u) - logical_col);
#else
    return logical_col;
#endif
}

void my_select_row(uint8_t row) {
    uint8_t hw_row = map_row(row);
    gpio_write(row_pins[hw_row], KB_MATRIX_ROW_ACTIVE_LEVEL);
}

uint8_t my_read_col(uint8_t col) {
    uint8_t hw_col = map_col(col);
    return gpio_read(col_pins[hw_col]);
}

void my_unselect_row(uint8_t row) {
    uint8_t hw_row = map_row(row);
    gpio_write(row_pins[hw_row], KB_MATRIX_ROW_IDLE_LEVEL);
}

int main(void) {
    static const char *key_names[4][4] = {
        {"K01", "K02", "K03", "K04"},
        {"K05", "K06", "K07", "K08"},
        {"K09", "K10", "K11", "K12"},
        {"K13", "K14", "K15", "K16"}
    };
    keyboard_control_t kb_ctl;
    keyboard_ops_t ops = {
        .matrix_select_row = my_select_row,
        .matrix_read_col = my_read_col,
        .matrix_unselect_row = my_unselect_row
    };
    keyboard_cb_t cb = { .on_event = on_key_event };

    if (keyboard_init(&kb_ctl, &ops, &cb) != KB_OK) {
        return -1;
    }

    for (uint8_t r = 0; r < 4; r++) {
        for (uint8_t c = 0; c < 4; c++) {
            uint16_t key_id = (uint16_t)(r * 4u + c);
            keyboard_register_matrix(r, c, key_names[r][c], key_id, &kb_ctl);
        }
    }

    while (1) {
        keyboard_poll(&kb_ctl, 10u);
        delay_ms(10);
    }
}
```

#### 3. Custom Backend Example

> Note: In `KB_BACKEND_CUSTOM`, `state_buf[i]` must map to the i-th registered key.
> And set `KB_BACKEND_MODE` to `KB_BACKEND_CUSTOM` in `keyboard_config.h`.

```c
#include <string.h>
#include "keyboard_driver.h"

#define CUSTOM_KEY_COUNT 3u

// Replace this with your real scanner result: 1=pressed, 0=released
static uint8_t hw_states[CUSTOM_KEY_COUNT] = {0u};

static int my_scan_snapshot(uint8_t *state_buf, uint16_t key_count) {
    if (state_buf == NULL || key_count != CUSTOM_KEY_COUNT) {
        return -1;
    }
    memcpy(state_buf, hw_states, CUSTOM_KEY_COUNT);
    return 0;
}

int main(void) {
    static const keyboard_key_cfg_t keys[CUSTOM_KEY_COUNT] = {
        {.keyname = "VOL_UP",   .key_id = 0x10u, .hw.hw_code = 0x100u},
        {.keyname = "VOL_DOWN", .key_id = 0x11u, .hw.hw_code = 0x101u},
        {.keyname = "MUTE",     .key_id = 0x12u, .hw.hw_code = 0x102u}
    };

    keyboard_control_t kb_ctl;
    keyboard_ops_t ops = {
        .scan_snapshot = my_scan_snapshot
    };
    keyboard_cb_t cb = {
        .on_event = on_key_event
    };

    if (keyboard_init(&kb_ctl, &ops, &cb) != KB_OK) {
        return -1;
    }

    for (uint8_t i = 0; i < CUSTOM_KEY_COUNT; i++) {
        if (keyboard_register_key(&keys[i], &kb_ctl) != KB_OK) {
            return -1;
        }
    }

    while (1) {
        keyboard_poll(&kb_ctl, 10u);
        delay_ms(10);
    }
}
```

### Configuration

Edit `keyboard_config.h` to customize:

```c
// Memory pool size (bytes)
#define KEYBOARD_POOL_SIZE 512u

// Maximum number of keys
#define KB_MAX_KEYS 16u

// Timing parameters (ms)
#define KB_DEBOUNCE_MS 20u
#define KB_LONGPRESS_MS 800u
#define KB_REPEAT_START_MS 500u
#define KB_REPEAT_PERIOD_MS 80u
#define KB_DOUBLE_CLICK_MS 250u

// Backend mode
#define KB_BACKEND_MODE KB_BACKEND_GPIO  // or KB_BACKEND_MATRIX / KB_BACKEND_CUSTOM

// Active level configuration
#define KB_GPIO_ACTIVE_LEVEL 1u
#define KB_MATRIX_ACTIVE_LEVEL 1u
#define KB_MATRIX_ROW_ACTIVE_LEVEL 1u
// Derived idle level for matrix row output
#define KB_MATRIX_ROW_IDLE_LEVEL ((KB_MATRIX_ROW_ACTIVE_LEVEL) ? 0u : 1u)

// Matrix orientation flags (used by your row/col mapping callbacks)
#define KB_MATRIX_ROW_REVERSE 0u
#define KB_MATRIX_COL_REVERSE 0u

// Matrix dimensions
#define KB_MATRIX_MAX_ROW 8u
#define KB_MATRIX_MAX_COL 8u
```

### API Reference

#### Initialization

```c
int keyboard_init(keyboard_control_t *ctl,
                  const keyboard_ops_t *ops,
                  const keyboard_cb_t *cb);
```

#### Key Registration

```c
// GPIO mode
int keyboard_register_gpio(uint8_t pin, const char *key_name,
                          uint16_t key_id, keyboard_control_t *ctl);

// Matrix mode
int keyboard_register_matrix(uint8_t row, uint8_t col,
                             const char *key_name, uint16_t key_id,
                             keyboard_control_t *ctl);

// Generic registration
int keyboard_register_key(const keyboard_key_cfg_t *cfg,
                         keyboard_control_t *ctl);
```

#### Polling

```c
void keyboard_poll(keyboard_control_t *ctl, uint32_t dt_ms);
```

Call this function periodically (recommended: 10ms interval). `dt_ms` is the elapsed time since the previous call.

### Event Types

| Event | Description |
|-------|-------------|
| `KB_EVT_PRESS` | Key pressed |
| `KB_EVT_RELEASE` | Key released |
| `KB_EVT_CLICK` | Single click (press + release) |
| `KB_EVT_DOUBLE_CLICK` | Double click |
| `KB_EVT_LONGPRESS` | Long press detected |
| `KB_EVT_LONGPRESS_RELEASE` | Long press released |
| `KB_EVT_REPEAT` | Auto-repeat event |

> Note: `KB_EVT_CLICK` is emitted after `KB_DOUBLE_CLICK_MS` timeout to avoid conflict with `KB_EVT_DOUBLE_CLICK`.

### Error Codes

| Code | Description |
|------|-------------|
| `KB_OK` | Success |
| `KB_ERR_PARAM` | Invalid parameter |
| `KB_ERR_BACKEND` | Backend capability mismatch |
| `KB_ERR_POOL_CFG` | Memory pool configuration error |
| `KB_ERR_RANGE` | Parameter out of range |
| `KB_ERR_DUPLICATE` | Duplicate key registration |
| `KB_ERR_FULL` | Maximum keys reached |
| `KB_ERR_NOMEM` | Memory allocation failed |

### Advanced Usage

#### Detecting Key Combinations

```c
#include <string.h>

#define MAX_PRESSED_KEYS 8u
#define KEY_CTRL 0x01u  // replace with your key_id
#define KEY_C    0x02u  // replace with your key_id
static uint16_t pressed_keys[MAX_PRESSED_KEYS];
static uint8_t pressed_count = 0u;

static int find_pressed_index(uint16_t key_id) {
    for (uint8_t i = 0; i < pressed_count; i++) {
        if (pressed_keys[i] == key_id) {
            return (int)i;
        }
    }
    return -1;
}

void on_key_event(const char *keyname, uint16_t key_id,
                  kb_event_t evt, void *user) {
    (void)keyname;
    (void)user;

    if (evt == KB_EVT_PRESS) {
        if (find_pressed_index(key_id) < 0 && pressed_count < MAX_PRESSED_KEYS) {
            pressed_keys[pressed_count++] = key_id;
        }

        if (find_pressed_index(KEY_CTRL) >= 0 && find_pressed_index(KEY_C) >= 0) {
            printf("Ctrl+C detected!\n");
        }
    } else if (evt == KB_EVT_RELEASE) {
        int idx = find_pressed_index(key_id);
        if (idx >= 0) {
            memmove(&pressed_keys[idx], &pressed_keys[idx + 1],
                    (pressed_count - (uint8_t)idx - 1u) * sizeof(pressed_keys[0]));
            pressed_count--;
        }
    }
}
```

#### Matrix Ghosting Note

Current matrix backend does **not** implement software anti-ghost filtering.  
If your matrix has no per-key diodes, 3+ key rectangle presses can still produce ghost keys.  
Use diode hardware, or add filtering in application logic / `KB_BACKEND_CUSTOM`.

### License

Apache-2.0

### Author

wsoz

### Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

## 中文

### 项目简介

一个面向嵌入式系统的生产级通用按键驱动框架。支持多种输入方式，包括独立GPIO按键、矩阵键盘和自定义扫描接口。具备完整的事件检测功能（单击、双击、长按、连发），内置防抖算法和高效内存管理。

### 核心特性

- **🎯 真正的硬件通用性**
  - 独立GPIO按键
  - 矩阵键盘（行列扫描）
  - 自定义扫描接口（I2C/SPI芯片等）

- **⚡ 丰富的事件检测**
  - 按下 / 释放
  - 单击 / 双击
  - 长按 / 长按释放
  - 自动连发

- **🛡️ 健壮的设计**
  - 软件防抖
  - 按键注册路径支持可选 lock/unlock
  - 多键同时检测
  - 支持矩阵原始扫描（未内置软件防鬼键）

- **💾 高效内存管理**
  - 自定义内存池（无碎片）
  - 可配置池大小
  - 支持静态分配

- **🔧 易于集成**
  - 清晰的API设计
  - 最小依赖
  - 完善的错误处理

### 架构设计

```
┌─────────────────────────────────────────┐
│           应用层                         │
│    (事件回调、组合键检测)                │
└─────────────────┬───────────────────────┘
                  │ 事件
┌─────────────────▼───────────────────────┐
│         按键驱动框架                     │
│  • 事件检测（单击/长按/连发）            │
│  • 防抖算法                              │
│  • 状态管理                              │
└─────────────────┬───────────────────────┘
                  │ 硬件抽象
┌─────────────────▼───────────────────────┐
│       硬件后端（用户实现）               │
│  • GPIO模式: read_pin()                 │
│  • 矩阵模式: select_row/read_col()      │
│  • 自定义: scan_snapshot()              │
└─────────────────────────────────────────┘
```

### 快速开始

#### 1. GPIO模式示例

```c
#include "keyboard_driver.h"

// 硬件层：GPIO读取函数
uint8_t my_read_pin(uint8_t pin) {
    return gpio_read(pin);  // 调用您的GPIO驱动
}

// 事件回调
void on_key_event(const char *keyname, uint16_t key_id,
                  kb_event_t evt, void *user) {
    switch (evt) {
        case KB_EVT_CLICK:
            printf("按键 %s 单击\n", keyname);
            break;
        case KB_EVT_LONGPRESS:
            printf("按键 %s 长按\n", keyname);
            break;
        // ... 处理其他事件
    }
}

int main(void) {
    keyboard_control_t kb_ctl;

    // 1. 配置硬件操作
    keyboard_ops_t ops = {
        .read_pin = my_read_pin,
        .get_tick_ms = NULL,  // 可选
        .lock = NULL,         // 可选
        .unlock = NULL
    };

    // 2. 配置回调
    keyboard_cb_t cb = {
        .on_event = on_key_event,
        .user = NULL
    };

    // 3. 初始化
    keyboard_init(&kb_ctl, &ops, &cb);

    // 4. 注册按键
    keyboard_register_gpio(0, "KEY_A", 0x01, &kb_ctl);
    keyboard_register_gpio(1, "KEY_B", 0x02, &kb_ctl);

    // 5. 主循环中轮询（每10ms）
    while (1) {
        keyboard_poll(&kb_ctl, 10);
        delay_ms(10);
    }
}
```

#### 2. 矩阵键盘示例

```c
#include "keyboard_driver.h"

static uint8_t row_pins[4] = {0, 1, 2, 3};  // 替换为你的行引脚
static uint8_t col_pins[4] = {4, 5, 6, 7};  // 替换为你的列引脚

static uint8_t map_row(uint8_t logical_row) {
#if KB_MATRIX_ROW_REVERSE
    return (uint8_t)((4u - 1u) - logical_row);
#else
    return logical_row;
#endif
}

static uint8_t map_col(uint8_t logical_col) {
#if KB_MATRIX_COL_REVERSE
    return (uint8_t)((4u - 1u) - logical_col);
#else
    return logical_col;
#endif
}

void my_select_row(uint8_t row) {
    uint8_t hw_row = map_row(row);
    gpio_write(row_pins[hw_row], KB_MATRIX_ROW_ACTIVE_LEVEL);
}

uint8_t my_read_col(uint8_t col) {
    uint8_t hw_col = map_col(col);
    return gpio_read(col_pins[hw_col]);
}

void my_unselect_row(uint8_t row) {
    uint8_t hw_row = map_row(row);
    gpio_write(row_pins[hw_row], KB_MATRIX_ROW_IDLE_LEVEL);
}

int main(void) {
    static const char *key_names[4][4] = {
        {"K01", "K02", "K03", "K04"},
        {"K05", "K06", "K07", "K08"},
        {"K09", "K10", "K11", "K12"},
        {"K13", "K14", "K15", "K16"}
    };
    keyboard_control_t kb_ctl;
    keyboard_ops_t ops = {
        .matrix_select_row = my_select_row,
        .matrix_read_col = my_read_col,
        .matrix_unselect_row = my_unselect_row
    };
    keyboard_cb_t cb = { .on_event = on_key_event };

    if (keyboard_init(&kb_ctl, &ops, &cb) != KB_OK) {
        return -1;
    }

    for (uint8_t r = 0; r < 4; r++) {
        for (uint8_t c = 0; c < 4; c++) {
            uint16_t key_id = (uint16_t)(r * 4u + c);
            keyboard_register_matrix(r, c, key_names[r][c], key_id, &kb_ctl);
        }
    }

    while (1) {
        keyboard_poll(&kb_ctl, 10u);
        delay_ms(10);
    }
}
```

#### 3. 自定义后端示例

> 注意：在 `KB_BACKEND_CUSTOM` 下，`state_buf[i]` 必须对应“第 i 个注册的按键”。
> 同时在 `keyboard_config.h` 里把 `KB_BACKEND_MODE` 设为 `KB_BACKEND_CUSTOM`。

```c
#include <string.h>
#include "keyboard_driver.h"

#define CUSTOM_KEY_COUNT 3u

// 替换为你的真实扫描结果：1=按下，0=释放
static uint8_t hw_states[CUSTOM_KEY_COUNT] = {0u};

static int my_scan_snapshot(uint8_t *state_buf, uint16_t key_count) {
    if (state_buf == NULL || key_count != CUSTOM_KEY_COUNT) {
        return -1;
    }
    memcpy(state_buf, hw_states, CUSTOM_KEY_COUNT);
    return 0;
}

int main(void) {
    static const keyboard_key_cfg_t keys[CUSTOM_KEY_COUNT] = {
        {.keyname = "VOL_UP",   .key_id = 0x10u, .hw.hw_code = 0x100u},
        {.keyname = "VOL_DOWN", .key_id = 0x11u, .hw.hw_code = 0x101u},
        {.keyname = "MUTE",     .key_id = 0x12u, .hw.hw_code = 0x102u}
    };

    keyboard_control_t kb_ctl;
    keyboard_ops_t ops = {
        .scan_snapshot = my_scan_snapshot
    };
    keyboard_cb_t cb = {
        .on_event = on_key_event
    };

    if (keyboard_init(&kb_ctl, &ops, &cb) != KB_OK) {
        return -1;
    }

    for (uint8_t i = 0; i < CUSTOM_KEY_COUNT; i++) {
        if (keyboard_register_key(&keys[i], &kb_ctl) != KB_OK) {
            return -1;
        }
    }

    while (1) {
        keyboard_poll(&kb_ctl, 10u);
        delay_ms(10);
    }
}
```

### 配置说明

编辑 `keyboard_config.h` 进行自定义：

```c
// 内存池大小（字节）
#define KEYBOARD_POOL_SIZE 512u

// 最大按键数量
#define KB_MAX_KEYS 16u

// 时间参数（毫秒）
#define KB_DEBOUNCE_MS 20u
#define KB_LONGPRESS_MS 800u
#define KB_REPEAT_START_MS 500u
#define KB_REPEAT_PERIOD_MS 80u
#define KB_DOUBLE_CLICK_MS 250u

// 后端模式
#define KB_BACKEND_MODE KB_BACKEND_GPIO  // 或 KB_BACKEND_MATRIX / KB_BACKEND_CUSTOM

// 有效电平配置
#define KB_GPIO_ACTIVE_LEVEL 1u
#define KB_MATRIX_ACTIVE_LEVEL 1u
#define KB_MATRIX_ROW_ACTIVE_LEVEL 1u
// 由行选通电平推导出的空闲电平
#define KB_MATRIX_ROW_IDLE_LEVEL ((KB_MATRIX_ROW_ACTIVE_LEVEL) ? 0u : 1u)

// 矩阵方向标志（在你的行列映射回调中使用）
#define KB_MATRIX_ROW_REVERSE 0u
#define KB_MATRIX_COL_REVERSE 0u

// 矩阵尺寸
#define KB_MATRIX_MAX_ROW 8u
#define KB_MATRIX_MAX_COL 8u
```

### API参考

#### 初始化

```c
int keyboard_init(keyboard_control_t *ctl,
                  const keyboard_ops_t *ops,
                  const keyboard_cb_t *cb);
```

#### 按键注册

```c
// GPIO模式
int keyboard_register_gpio(uint8_t pin, const char *key_name,
                          uint16_t key_id, keyboard_control_t *ctl);

// 矩阵模式
int keyboard_register_matrix(uint8_t row, uint8_t col,
                             const char *key_name, uint16_t key_id,
                             keyboard_control_t *ctl);

// 通用注册
int keyboard_register_key(const keyboard_key_cfg_t *cfg,
                         keyboard_control_t *ctl);
```

#### 轮询

```c
void keyboard_poll(keyboard_control_t *ctl, uint32_t dt_ms);
```

定期调用此函数（推荐：10ms间隔）。`dt_ms` 表示距离上一次调用的时间增量（毫秒）。

### 事件类型

| 事件 | 说明 |
|------|------|
| `KB_EVT_PRESS` | 按键按下 |
| `KB_EVT_RELEASE` | 按键释放 |
| `KB_EVT_CLICK` | 单击（按下+释放） |
| `KB_EVT_DOUBLE_CLICK` | 双击 |
| `KB_EVT_LONGPRESS` | 检测到长按 |
| `KB_EVT_LONGPRESS_RELEASE` | 长按释放 |
| `KB_EVT_REPEAT` | 自动连发事件 |

> 注意：为避免与 `KB_EVT_DOUBLE_CLICK` 冲突，`KB_EVT_CLICK` 会在 `KB_DOUBLE_CLICK_MS` 超时后才触发。

### 错误码

| 代码 | 说明 |
|------|------|
| `KB_OK` | 成功 |
| `KB_ERR_PARAM` | 参数无效 |
| `KB_ERR_BACKEND` | 后端能力不匹配 |
| `KB_ERR_POOL_CFG` | 内存池配置错误 |
| `KB_ERR_RANGE` | 参数超出范围 |
| `KB_ERR_DUPLICATE` | 重复注册按键 |
| `KB_ERR_FULL` | 达到最大按键数 |
| `KB_ERR_NOMEM` | 内存分配失败 |

### 高级用法

#### 检测组合键

```c
#include <string.h>

#define MAX_PRESSED_KEYS 8u
#define KEY_CTRL 0x01u  // 按你的 key_id 定义
#define KEY_C    0x02u  // 按你的 key_id 定义
static uint16_t pressed_keys[MAX_PRESSED_KEYS];
static uint8_t pressed_count = 0u;

static int find_pressed_index(uint16_t key_id) {
    for (uint8_t i = 0; i < pressed_count; i++) {
        if (pressed_keys[i] == key_id) {
            return (int)i;
        }
    }
    return -1;
}

void on_key_event(const char *keyname, uint16_t key_id,
                  kb_event_t evt, void *user) {
    (void)keyname;
    (void)user;

    if (evt == KB_EVT_PRESS) {
        if (find_pressed_index(key_id) < 0 && pressed_count < MAX_PRESSED_KEYS) {
            pressed_keys[pressed_count++] = key_id;
        }

        if (find_pressed_index(KEY_CTRL) >= 0 && find_pressed_index(KEY_C) >= 0) {
            printf("检测到 Ctrl+C!\n");
        }
    } else if (evt == KB_EVT_RELEASE) {
        int idx = find_pressed_index(key_id);
        if (idx >= 0) {
            memmove(&pressed_keys[idx], &pressed_keys[idx + 1],
                    (pressed_count - (uint8_t)idx - 1u) * sizeof(pressed_keys[0]));
            pressed_count--;
        }
    }
}
```

#### 矩阵鬼键说明

当前矩阵后端**未内置软件防鬼键算法**。  
如果硬件没有逐键二极管，3 键及以上构成矩形时仍可能出现鬼键。  
建议使用逐键二极管，或在应用层 / `KB_BACKEND_CUSTOM` 中补充过滤策略。

### 许可证

Apache-2.0

### 作者

wsoz

### 贡献

欢迎贡献！请随时提交 Pull Request。
