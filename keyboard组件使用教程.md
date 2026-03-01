# keyboard组件使用教程

> 本文档将介绍一下如何快速使用本组件进行辅助开发
>
> ​																		------wsoz



## 配置页修改

在进行GPIO按键操作时，我们首先需要去` keyboard_config.h` 中切换一下我们的` KB_BACKEND_MODE` 模式以适配我们的矩阵还是GPIO按键。

```c
/* 默认使用矩阵键盘，可在工程配置里覆写 */
#ifndef KB_BACKEND_MODE
#define KB_BACKEND_MODE KB_BACKEND_GPIO

//同时注意在我们的那个按键控制块中选择
g_key_ctl.backend_mode=KB_BACKEND_MODE;
```

## 初始化

在初始化中我们主要就是将ops操作集，回调函数等绑定到`keyboard_control_t`按键控制结构体中

### **按键注册描述符初始化**

```c
typedef struct
{
    const char *keyname;      /* 逻辑名称，如 "K_A" */
    uint16_t key_id;          /* 逻辑按键ID，业务层推荐用这个 */
    keyboard_hw_ref_t hw;     /* 硬件定位信息 */
} keyboard_key_cfg_t;

/* 硬件定位：独立 GPIO / 矩阵 row-col / 自定义编码 */
typedef union
{
    uint8_t gpio_pin;
    keyboard_matrix_pos_t matrix;
    uint16_t hw_code;
} keyboard_hw_ref_t;

/* 矩阵键盘位置 */
typedef struct
{
    uint8_t row;
    uint8_t col;
} keyboard_matrix_pos_t;

keyboard_key_cfg_t cfg={0};
```

争对GPIO按键以及矩阵按键初始化不同

```c
/*通用*/
cfg.keyname="name";
cfg.key_id=pin_id;	//此处即是按键的一个区分符,逻辑标识应用层回调解析使用

/*GPIO*/
cfg.hw.gpio_pin=pin_num;	//这个按键对应的“硬件引脚编号”，驱动层去读取的引脚
/* 4*4为例子矩阵键盘 */	行列坐标
static const char *name_4x4[4][4] = {
    {"K01","K02","K03","K04"},
    {"K05","K06","K07","K08"},
    {"K09","K10","K11","K12"},
    {"K13","K14","K15","K16"},
};

for (uint8_t r = 0; r < 4; r++) {
    for (uint8_t c = 0; c < 4; c++) {
        keyboard_key_cfg_t cfg = {0};
        cfg.keyname = name_4x4[r][c];
        cfg.key_id = (uint16_t)(r * 4u + c + 1u);
        cfg.hw.matrix.row = r;
        cfg.hw.matrix.col = c;
        keyboard_register_key(&cfg, &g_key_ctl);
    }
}

/* 推荐使用便捷接口 */
static const char *name_4x4[4][4] = {
    {"K01","K02","K03","K04"},
    {"K05","K06","K07","K08"},
    {"K09","K10","K11","K12"},
    {"K13","K14","K15","K16"},
};

for (uint8_t r = 0; r < 4; r++) {
    for (uint8_t c = 0; c < 4; c++) {
        uint16_t key_id = (uint16_t)(r * 4u + c + 1u); // 1~16
        keyboard_register_matrix(r, c, name_4x4[r][c], key_id, &g_key_ctl);
    }
}
```

### **硬件操作集初始化**

接下来就需要去对接我们的ops操作集合

```c
/* keyboard 操作集 */
typedef struct
{
    /* GPIO 后端：读取 pin 电平，返回 0/1 */
    uint8_t (*read_pin)(uint8_t pin);

    /* 矩阵后端：驱动行并读取列 */
    void (*matrix_select_row)(uint8_t row);
    uint8_t (*matrix_read_col)(uint8_t col);
    void (*matrix_unselect_row)(uint8_t row);

    /*
     * 自定义后端（复杂输入建议使用）：
     * 按“注册顺序”输出 key_count 个按键电平到 state_buf（每个元素取值0/1）
     * 返回 0 表示成功
     */
    int (*scan_snapshot)(uint8_t *state_buf, uint16_t key_count);

    /* 获取当前毫秒 tick（可选，不提供则可以依赖 poll 的 dt_ms） */
    uint32_t (*get_tick_ms)(void);

    /* 可选：多线程环境保护 */
    void (*lock)(void);
    void (*unlock)(void);
} keyboard_ops_t;

keyboard_ops_t ops={0};
```

下面争对我们的GPIO以及矩阵键盘进行讲解

```c
/*GPIO*/
ops.read_pin=read_pin_level;	//注意需要和函数指针声明相同

/*矩阵键盘*/
示例（RT-Thread）：

static const rt_base_t row_pins[4] = { GET_PIN(E,9), GET_PIN(E,10),
GET_PIN(E,11), GET_PIN(E,12) };
static const rt_base_t col_pins[4] = { GET_PIN(B,0), GET_PIN(B,1), GET_PIN(E,7),
GET_PIN(E,8)  };

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
    .matrix_unselect_row = kb_unselect_row
        // 下面可选
    .lock = RT_NULL,
    .unlock = RT_NULL
    .get_tick_ms = RT_NULL,
}
```

### 回调函数初始化

我们的应用层主要通过回调函数来处理对应的事件完成按键任务。

```c
typedef struct
{
    keyboard_event_cb on_event;
    void *user;
} keyboard_cb_t;

keyboard_cb_t cb={0}；
```

按键的回调函数定义格式：

```c
static void kb_event_cb(const char *keyname, uint16_t key_id, kb_event_t evt,void *user)
{
    rt_kprintf("key=%s id=%d evt=%d\n", keyname, key_id, evt);
}
cb.on_event=kb_event_cb;
```

在此回调函数中我们就可以通过`switch-case`等来对`key_id`以及`evt`进行分层处理,具体事件的定义可以去.h文件中参考`kb_event_t`。



### 系统挂载初始化

我们在完成上述操作之后就已经完成了那个基础的按键定义了，现在要做的就是完成将我们已经定义了按键挂载在我们的整个按键控制块上。

```c
/*首先将ops操作集以及回调函数挂载到控制结构体*/
keyboard_init(&g_key_ctl, &ops, &cb);

/*然后将cfg按键描述挂载到控制结构体*/
keyboard_register_key(&cfg,&g_key_ctl);
```



## 系统运行

按键初始化完成之后我们就可以运行我们的按键控制系统了。

```c
/*运行按键*/
keyboard_poll(&g_key_ctl, 10u)
//注意:第二个参数即是我们实际调用轮询的周期以stm32hal库为例子
static uint32_t tick=hal_get_tick();
static uint32_t last_tick;
if(tick-last_tick>dt)
{
	keyboard_poll(&g_key_ctl, dt);
    last_tick=tick;
}
```



## TIPS

如果对按键的灵敏性有要求的可以去`config`中手动修改阈值时间等。