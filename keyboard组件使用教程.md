# keyboard组件使用教程

> 本文档用于快速上手 keyboard 组件（GPIO / 矩阵 / 自定义后端）
>
> -- wsoz

## 1. 配置修改（keyboard_config.h）

先在 `keyboard_config.h` 里选择后端模式：

```c
/* 采集后端模式 */
#define KB_BACKEND_GPIO   1u
#define KB_BACKEND_MATRIX 2u
#define KB_BACKEND_CUSTOM 3u

/* 默认使用矩阵键盘，可在工程配置里覆写 */
#ifndef KB_BACKEND_MODE
#define KB_BACKEND_MODE KB_BACKEND_GPIO
#endif
```

注意：`g_key_ctl.backend_mode` 不需要手动赋值，`keyboard_init()` 内部会根据 `KB_BACKEND_MODE` 自动设置。

---

## 2. 初始化总流程

统一流程如下：

1. 配置硬件 IO（GPIO 模式或矩阵行列模式）
2. 填充 `keyboard_ops_t` 和 `keyboard_cb_t`
3. 调用 `keyboard_init(&ctl, &ops, &cb)`
4. 注册按键（`keyboard_register_gpio/matrix` 或 `keyboard_register_key`）
5. 周期调用 `keyboard_poll(&ctl, dt_ms)`

---

## 3. 按键描述符（keyboard_key_cfg_t）

```c
typedef struct
{
    const char *keyname;      /* 逻辑名称，如 "K_A" */
    uint16_t key_id;          /* 逻辑按键ID，应用层推荐用这个 */
    keyboard_hw_ref_t hw;     /* 硬件定位信息 */
} keyboard_key_cfg_t;

typedef union
{
    uint8_t gpio_pin;         /* GPIO 模式 */
    keyboard_matrix_pos_t matrix; /* 矩阵模式 */
    uint16_t hw_code;         /* 自定义模式 */
} keyboard_hw_ref_t;
```

- GPIO 模式：只填 `cfg.hw.gpio_pin`
- 矩阵模式：只填 `cfg.hw.matrix.row/col`
- 自定义模式：只填 `cfg.hw.hw_code`

---

## 4. 操作集（keyboard_ops_t）

```c
keyboard_ops_t ops = {0};
```

### 4.1 GPIO 模式

```c
static uint8_t read_pin_level(uint8_t pin)
{
    return (uint8_t)rt_pin_read((rt_base_t)pin);
}

ops.read_pin = read_pin_level;
```

### 4.2 矩阵模式（RT-Thread 示例）

```c
static const rt_base_t row_pins[4] = {
    GET_PIN(E,9), GET_PIN(E,10), GET_PIN(E,11), GET_PIN(E,12)
};

static const rt_base_t col_pins[4] = {
    GET_PIN(B,0), GET_PIN(B,1), GET_PIN(E,7), GET_PIN(E,8)
};

static void kb_select_row(uint8_t row)
{
    rt_pin_write(row_pins[row], KB_MATRIX_ROW_ACTIVE_LEVEL);
}

static uint8_t kb_read_col(uint8_t col)
{
    return (uint8_t)rt_pin_read(col_pins[col]);
}

static void kb_unselect_row(uint8_t row)
{
    rt_pin_write(row_pins[row], KB_MATRIX_ROW_IDLE_LEVEL);
}

keyboard_ops_t ops = {
    .matrix_select_row   = kb_select_row,
    .matrix_read_col     = kb_read_col,
    .matrix_unselect_row = kb_unselect_row,
    .lock                = RT_NULL,   /* 可选 */
    .unlock              = RT_NULL,   /* 可选 */
    .get_tick_ms         = RT_NULL    /* 可选 */
};
```

---

## 5. 回调函数（keyboard_cb_t）

```c
static void kb_event_cb(const char *keyname, uint16_t key_id, kb_event_t evt, void *user)
{
    rt_kprintf("key=%s id=%d evt=%d\n", keyname, key_id, evt);
}

keyboard_cb_t cb = {0};
cb.on_event = kb_event_cb;
cb.user = RT_NULL;
```

---

## 6. 注册与挂载

### 6.1 GPIO 单按键示例

```c
keyboard_control_t g_key_ctl = {0};
keyboard_ops_t ops = {0};
keyboard_cb_t cb = {0};

rt_base_t pin_num = rt_pin_get("PA.0");
rt_pin_mode(pin_num, PIN_MODE_INPUT_PULLDOWN); /* 按下高电平时常用 */

ops.read_pin = read_pin_level;
cb.on_event = kb_event_cb;

keyboard_init(&g_key_ctl, &ops, &cb);
keyboard_register_gpio((uint8_t)pin_num, "K1", 1u, &g_key_ctl);
```

### 6.2 4x4 矩阵示例

```c
static const char *name_4x4[4][4] = {
    {"K01","K02","K03","K04"},
    {"K05","K06","K07","K08"},
    {"K09","K10","K11","K12"},
    {"K13","K14","K15","K16"},
};

keyboard_control_t g_key_ctl = {0};
keyboard_ops_t ops = {0};
keyboard_cb_t cb = {0};

/* 1) 先初始化行列IO */
for (uint8_t r = 0; r < 4; r++) {
    rt_pin_mode(row_pins[r], PIN_MODE_OUTPUT);
    rt_pin_write(row_pins[r], KB_MATRIX_ROW_IDLE_LEVEL);
}

#if (KB_MATRIX_ACTIVE_LEVEL == 1u)
for (uint8_t c = 0; c < 4; c++) {
    rt_pin_mode(col_pins[c], PIN_MODE_INPUT_PULLDOWN);
}
#else
for (uint8_t c = 0; c < 4; c++) {
    rt_pin_mode(col_pins[c], PIN_MODE_INPUT_PULLUP);
}
#endif

/* 2) ops/cb */
ops.matrix_select_row = kb_select_row;
ops.matrix_read_col = kb_read_col;
ops.matrix_unselect_row = kb_unselect_row;
cb.on_event = kb_event_cb;

/* 3) init */
keyboard_init(&g_key_ctl, &ops, &cb);

/* 4) register */
for (uint8_t r = 0; r < 4; r++) {
    for (uint8_t c = 0; c < 4; c++) {
        uint16_t key_id = (uint16_t)(r * 4u + c + 1u);
        keyboard_register_matrix(r, c, name_4x4[r][c], key_id, &g_key_ctl);
    }
}
```

> 先 `keyboard_init`，再注册按键；不要在矩阵循环注册后再额外注册一个未填完整的 `cfg`。

---

## 7. 系统运行（轮询）

### 7.1 固定周期（推荐）

```c
while (1)
{
    keyboard_poll(&g_key_ctl, 10u);
    rt_thread_mdelay(10);
}
```

### 7.2 非固定周期（裸机常见）

```c
uint32_t last_tick = hal_get_tick();
while (1)
{
    uint32_t now = hal_get_tick();
    uint32_t dt = now - last_tick;
    if (dt > 0u)
    {
        keyboard_poll(&g_key_ctl, dt);
        last_tick = now;
    }
}
```

`dt_ms` 的含义是“距离上一次 `keyboard_poll` 的时间增量（ms）”。

---

## 8. 常见坑（建议先看）

1. `ops` / `cb` 是结构体，不是函数指针本体：
   - 正确：`ops.read_pin = read_pin_level;`
   - 正确：`cb.on_event = kb_event_cb;`
2. GPIO pin 请用 `rt_pin_get("PA.0")` 的返回值，不要手写常量。
3. 矩阵模式要先配置行列 GPIO 方向和上下拉。
4. `KB_EVT_CLICK` 为避免和双击冲突，会在 `KB_DOUBLE_CLICK_MS` 超时后发出。
5. 当前矩阵后端未内置软件防鬼键；无二极管矩阵在多键下可能出现鬼键。

---

## 9. TIPS

若对手感有要求，可在 `keyboard_config.h` 调整以下参数：

- `KB_DEBOUNCE_MS`
- `KB_LONGPRESS_MS`
- `KB_REPEAT_START_MS`
- `KB_REPEAT_PERIOD_MS`
- `KB_DOUBLE_CLICK_MS`
