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
  - Hardware debouncing
  - Thread-safe (optional lock/unlock)
  - Ghost key prevention (matrix mode)
  - Multiple keys simultaneous detection

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
// Hardware layer: Matrix scan functions
void my_select_row(uint8_t row) {
    gpio_write(row_pins[row], 1);  // Pull row high
}

uint8_t my_read_col(uint8_t col) {
    return gpio_read(col_pins[col]);
}

void my_unselect_row(uint8_t row) {
    gpio_write(row_pins[row], 0);  // Pull row low
}

int main(void) {
    keyboard_control_t kb_ctl;

    keyboard_ops_t ops = {
        .matrix_select_row = my_select_row,
        .matrix_read_col = my_read_col,
        .matrix_unselect_row = my_unselect_row
    };

    keyboard_cb_t cb = { .on_event = on_key_event };

    keyboard_init(&kb_ctl, &ops, &cb);

    // Register 4x4 matrix keys
    for (uint8_t r = 0; r < 4; r++) {
        for (uint8_t c = 0; c < 4; c++) {
            uint16_t key_id = r * 4 + c;
            char name[16];
            sprintf(name, "K_%d_%d", r, c);
            keyboard_register_matrix(r, c, name, key_id, &kb_ctl);
        }
    }

    while (1) {
        keyboard_poll(&kb_ctl, 10);
        delay_ms(10);
    }
}
```

### Configuration

Edit `keyboard_config.h` to customize:

```c
// Maximum number of keys
#define KB_MAX_KEYS 16u

// Timing parameters (ms)
#define KB_DEBOUNCE_MS 20u          // Debounce time
#define KB_LONGPRESS_MS 800u        // Long-press threshold
#define KB_REPEAT_START_MS 500u     // Repeat start delay
#define KB_REPEAT_PERIOD_MS 80u     // Repeat interval
#define KB_DOUBLE_CLICK_MS 250u     // Double-click window

// Backend mode
#define KB_BACKEND_MODE KB_BACKEND_GPIO    // or KB_BACKEND_MATRIX
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

Call this function periodically (recommended: 10ms interval).

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
static uint16_t pressed_keys[8];
static uint8_t pressed_count = 0;

void on_key_event(const char *keyname, uint16_t key_id,
                  kb_event_t evt, void *user) {
    if (evt == KB_EVT_PRESS) {
        pressed_keys[pressed_count++] = key_id;

        // Check for Ctrl+C (example)
        if (pressed_count == 2) {
            if ((pressed_keys[0] == KEY_CTRL && pressed_keys[1] == KEY_C) ||
                (pressed_keys[0] == KEY_C && pressed_keys[1] == KEY_CTRL)) {
                printf("Ctrl+C detected!\n");
            }
        }
    } else if (evt == KB_EVT_RELEASE) {
        // Remove from pressed_keys array
        for (uint8_t i = 0; i < pressed_count; i++) {
            if (pressed_keys[i] == key_id) {
                memmove(&pressed_keys[i], &pressed_keys[i+1],
                       (pressed_count - i - 1) * sizeof(uint16_t));
                pressed_count--;
                break;
            }
        }
    }
}
```

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
  - 硬件防抖
  - 线程安全（可选锁机制）
  - 鬼键预防（矩阵模式）
  - 多键同时检测

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
// 硬件层：矩阵扫描函数
void my_select_row(uint8_t row) {
    gpio_write(row_pins[row], 1);  // 拉高行
}

uint8_t my_read_col(uint8_t col) {
    return gpio_read(col_pins[col]);
}

void my_unselect_row(uint8_t row) {
    gpio_write(row_pins[row], 0);  // 拉低行
}

int main(void) {
    keyboard_control_t kb_ctl;

    keyboard_ops_t ops = {
        .matrix_select_row = my_select_row,
        .matrix_read_col = my_read_col,
        .matrix_unselect_row = my_unselect_row
    };

    keyboard_cb_t cb = { .on_event = on_key_event };

    keyboard_init(&kb_ctl, &ops, &cb);

    // 注册4x4矩阵按键
    for (uint8_t r = 0; r < 4; r++) {
        for (uint8_t c = 0; c < 4; c++) {
            uint16_t key_id = r * 4 + c;
            char name[16];
            sprintf(name, "K_%d_%d", r, c);
            keyboard_register_matrix(r, c, name, key_id, &kb_ctl);
        }
    }

    while (1) {
        keyboard_poll(&kb_ctl, 10);
        delay_ms(10);
    }
}
```

### 配置说明

编辑 `keyboard_config.h` 进行自定义：

```c
// 最大按键数量
#define KB_MAX_KEYS 16u

// 时间参数（毫秒）
#define KB_DEBOUNCE_MS 20u          // 防抖时间
#define KB_LONGPRESS_MS 800u        // 长按阈值
#define KB_REPEAT_START_MS 500u     // 连发启动延迟
#define KB_REPEAT_PERIOD_MS 80u     // 连发间隔
#define KB_DOUBLE_CLICK_MS 250u     // 双击时间窗口

// 后端模式
#define KB_BACKEND_MODE KB_BACKEND_GPIO    // 或 KB_BACKEND_MATRIX
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

定期调用此函数（推荐：10ms间隔）。

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
static uint16_t pressed_keys[8];
static uint8_t pressed_count = 0;

void on_key_event(const char *keyname, uint16_t key_id,
                  kb_event_t evt, void *user) {
    if (evt == KB_EVT_PRESS) {
        pressed_keys[pressed_count++] = key_id;

        // 检测 Ctrl+C（示例）
        if (pressed_count == 2) {
            if ((pressed_keys[0] == KEY_CTRL && pressed_keys[1] == KEY_C) ||
                (pressed_keys[0] == KEY_C && pressed_keys[1] == KEY_CTRL)) {
                printf("检测到 Ctrl+C!\n");
            }
        }
    } else if (evt == KB_EVT_RELEASE) {
        // 从 pressed_keys 数组中移除
        for (uint8_t i = 0; i < pressed_count; i++) {
            if (pressed_keys[i] == key_id) {
                memmove(&pressed_keys[i], &pressed_keys[i+1],
                       (pressed_count - i - 1) * sizeof(uint16_t));
                pressed_count--;
                break;
            }
        }
    }
}
```

### 许可证

Apache-2.0

### 作者

wsoz

### 贡献

欢迎贡献！请随时提交 Pull Request。
