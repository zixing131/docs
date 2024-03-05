

# USB 设备开发：从入门到实践指南（三）

**作者：Hcamael@知道创宇 404 实验室  
时间：2024 年 2 月 29 日**

经过[上一篇文章](https://paper.seebug.org/3123/ "上一篇文章")的学习，对 USB HID 驱动有了更多的了解，但是也产生了许多疑问，在后续的学习中解决了一些疑问，本篇文章先对已经解决的问题进行讲解。

# 1 Nintendo 手柄驱动

在上一篇文章中，提到过一个问题：Switch 原装手柄没有 USB 接口，为什么在`nintendo`中还要进行一些列处理？

在后续的研究中发现了这个问题的答案：首先查看`hid-nintendo.c`驱动代码中`nintendo_hid_devices`结构体的定义，如下所示：

```c
static const struct hid_device_id nintendo_hid_devices[] = {
    { HID_USB_DEVICE(USB_VENDOR_ID_NINTENDO,
             USB_DEVICE_ID_NINTENDO_PROCON) },
    { HID_BLUETOOTH_DEVICE(USB_VENDOR_ID_NINTENDO,
             USB_DEVICE_ID_NINTENDO_PROCON) },
    { HID_USB_DEVICE(USB_VENDOR_ID_NINTENDO,
             USB_DEVICE_ID_NINTENDO_CHRGGRIP) },
    { HID_BLUETOOTH_DEVICE(USB_VENDOR_ID_NINTENDO,
             USB_DEVICE_ID_NINTENDO_JOYCONL) },
    { HID_BLUETOOTH_DEVICE(USB_VENDOR_ID_NINTENDO,
             USB_DEVICE_ID_NINTENDO_JOYCONR) },
    { }
};
```

仔细查看该结构体可以发现，对于 Switch 原装的左右手柄，使用的是`HID_BLUETOOTH_DEVICE`宏定义，表示匹配的是蓝牙 HID 协议，并不匹配 USB HID 协议。这样，该问题的答案就很明显了，`hid-nintendo.c`驱动适配了`JOYCONR`和`JOYCONL`手柄的蓝牙驱动。

第二个问题：为什么上一篇文章中模拟的 Switch Pro 手柄只创建了`/dev/input/eventX`却没有`/dev/input/jsX`？

关于该问题，我们就需要加深一点对内核 input 驱动的了解。

# 2 Linux 内核 INPUT 子系统简述

因为暂时没有开发 input 相关驱动的打算，所以并不会深入讲解 input 驱动的各项细节，本章节的目标是让读者读完以后，在心中能对 input 驱动的运作模式有个大致的了解。

## 2.1 注册 input event

用`hid-nintendo.c`驱动作为例子进行讲解，首先看`nintendo_hid_probe`函数，在上一篇文章中说过，当 USB HID 设备注册成功后，会在内核中匹配所有`.id_tables`，当匹配到`idVendor`和`idProduct`为`Switch Pro`手柄后，会调用`nintendo_hid_probe`函数。

接着就是进行一系列的初始化，还会和手柄进行一些通信，或者手柄的一些信息，这期间没出错的话就会调用到`joycon_input_create`函数，在该函数中通过`input_set_abs_params`，`input_set_capability`这类 input 驱动的函数，设置设备有哪些属性，比如有`EV_KEY`，就是表示该设备有按键，使用`input_set_abs_params`设置的就是坐标系，比如手柄的摇杆，鼠标的移动都需要使用该函数。

最后再调用`input_register_device`函数，如果没有意外，一个 input 事件就注册成功了，我们就可以通过`/dev/input/eventX`文件来进行通信，上一篇文章中提过，`eventX`文件的结构体如下所示：

```c
struct input_event {
    struct timeval time;
    __u16 type;
    __u16 code;
    __s32 value;
};
```

如果我们按了手柄的一个按键，那么这个时候读取`eventX`进行解析，我们会发现`type`的值就是`EV_KEY`，而`code`的值表示的就是某个按键，`value`表示的就是`1`或`0`（按下或者释放）。

我们能获取到的值同样也可以在`hid-nintendo.c`驱动中看到实现的代码，可以查看`nintendo_hid_event`函数，该函数为当接收到数据后，会调用的函数。

在`nintendo_hid_probe`函数中，当成功注册完 input 事件后，会设置一个状态：`ctlr->ctlr_state = JOYCON_CTLR_STATE_READ;`

在`nintendo_hid_event->joycon_ctlr_handle_event`函数会判断`ctlr->ctlr_state`的值如果等于`JOYCON_CTLR_STATE_READ`时，会调用`ret = joycon_ctlr_read_handler(ctlr, data, size);`函数。

该函数就是处理手柄的输入（按键，摇杆）数据的主函数，接着通过 input 驱动的`input_report_abs`,`input_report_key`这类的函数对坐标的状态，按键的状态进行设置，最后调用`input_sync`函数，就会把设置好的手柄输入传送到`/dev/input/eventX`文件中，我们通过`eventX`文件读取到的内容就是这么产生的。

从上面的内容可以知道，如果想要开发 Linux 下的`Switch Pro`手柄的客户端，只需要操作`eventX`文件，并且仔细阅读`nintendo_hid_event`函数，了解传输数据的数据结构就能实现。

## 2.2 注册手柄驱动

目前 Linux 下绝大部分手柄的客户端程序都是通过读取`/dev/input/jsX`文件获取手柄输入的数据，在上一篇 Paper 中，我们模拟的 XBox 手柄就能成功生成`/dev/input/jsX`驱动文件，但是`Swtich Pro`手柄却无法生成手柄驱动文件，这是为什么呢？

经过研究发现，控制生成手柄驱动的代码位于`drivers/input/joydev.c`文件中，对于驱动文件，首先关注的入口点同样是`input_handler`结构体，`joydev.c`驱动的`input_handler`结构体如下所示：

```c
static struct input_handler joydev_handler = {
    .event      = joydev_event,
    .match      = joydev_match,
    .connect    = joydev_connect,
    .disconnect = joydev_disconnect,
    .legacy_minors  = true,
    .minor      = JOYDEV_MINOR_BASE,
    .name       = "joydev",
    .id_table   = joydev_ids,
};
```

并且在`joydev_connect`函数中可以发现代码：`dev_set_name(&joydev->dev, "js%d", dev_no);`。

`joydev.c`驱动实现细节请自行阅读代码，下面来概括一下手柄驱动的大致流程：

1.跟其他驱动一样，当驱动被加载到内核当中后，会把`.id_table`加入到内核的`id_table`链表中。  
2.当一个新的`input event`被注册成功，会对`id_table`进行遍历，匹配合适的驱动。  
3.参见`joydev.c`的`joydev_ids`代码，如下所示：

```c
static const struct input_device_id joydev_ids[] = {
    {
        .flags = INPUT_DEVICE_ID_MATCH_EVBIT |
                INPUT_DEVICE_ID_MATCH_ABSBIT,
        .evbit = { BIT_MASK(EV_ABS) },
        .absbit = { BIT_MASK(ABS_X) },
    },
    {
        .flags = INPUT_DEVICE_ID_MATCH_EVBIT |
                INPUT_DEVICE_ID_MATCH_ABSBIT,
        .evbit = { BIT_MASK(EV_ABS) },
        .absbit = { BIT_MASK(ABS_Z) },
    },
    {
        .flags = INPUT_DEVICE_ID_MATCH_EVBIT |
                INPUT_DEVICE_ID_MATCH_ABSBIT,
        .evbit = { BIT_MASK(EV_ABS) },
        .absbit = { BIT_MASK(ABS_WHEEL) },
    },
    {
        .flags = INPUT_DEVICE_ID_MATCH_EVBIT |
                INPUT_DEVICE_ID_MATCH_ABSBIT,
        .evbit = { BIT_MASK(EV_ABS) },
        .absbit = { BIT_MASK(ABS_THROTTLE) },
    },
    {
        .flags = INPUT_DEVICE_ID_MATCH_EVBIT |
                INPUT_DEVICE_ID_MATCH_KEYBIT,
        .evbit = { BIT_MASK(EV_KEY) },
        .keybit = {[BIT_WORD(BTN_JOYSTICK)] = BIT_MASK(BTN_JOYSTICK) },
    },
    {
        .flags = INPUT_DEVICE_ID_MATCH_EVBIT |
                INPUT_DEVICE_ID_MATCH_KEYBIT,
        .evbit = { BIT_MASK(EV_KEY) },
        .keybit = { [BIT_WORD(BTN_GAMEPAD)] = BIT_MASK(BTN_GAMEPAD) },
    },
    {
        .flags = INPUT_DEVICE_ID_MATCH_EVBIT |
                INPUT_DEVICE_ID_MATCH_KEYBIT,
        .evbit = { BIT_MASK(EV_KEY) },
        .keybit = { [BIT_WORD(BTN_TRIGGER_HAPPY)] = BIT_MASK(BTN_TRIGGER_HAPPY) },
    },
    { } /* Terminating entry */
};
```

从上面代码可以看出当`input event`设置了`EV_ABS`坐标系或者`EV_KEY`按键属性的时候，就能匹配到该驱动。

4.匹配到该驱动后，将会调用`joydev_connect`函数，经过一系列初始化，然后创建`/dev/input/jsX`驱动文件。  
5.当一个新的`event`产生，将会调用`joydev_event`函数，首先查看该函数的定义，代码如下所示：

```c
static void joydev_event(struct input_handle *handle,
             unsigned int type, unsigned int code, int value)
{
    struct joydev *joydev = handle->private;
    struct joydev_client *client;
    struct js_event event;

    switch (type) {

    case EV_KEY:
        if (code < BTN_MISC || value == 2)
            return;
        event.type = JS_EVENT_BUTTON;
        event.number = joydev->keymap[code - BTN_MISC];
        event.value = value;
        break;

    case EV_ABS:
        event.type = JS_EVENT_AXIS;
        event.number = joydev->absmap[code];
        event.value = joydev_correct(value,
                    &joydev->corr[event.number]);
        if (event.value == joydev->abs[event.number])
            return;
        joydev->abs[event.number] = event.value;
        break;

    default:
        return;
    }

    event.time = jiffies_to_msecs(jiffies);

    rcu_read_lock();
    list_for_each_entry_rcu(client, &joydev->client_list, node)
        joydev_pass_event(client, &event);
    rcu_read_unlock();

    wake_up_interruptible(&joydev->wait);
}
```

`joydev_event`的三个参数正好能和`input_event`结构体对应，接着将会根据`input_event`结构体的数据生成`js_event`结构体。

在上一篇文章中，讲述的读取`/dev/input/jsX`的数据，正好能和上面的代码对应上。

在了解了`joydev.c`驱动的加载流程后，也并没有解决我们之前提出的疑问：为什么`Switch Pro`手柄不能加载`joydev`驱动呢？

因为还有一个`joydev_match`函数，相关代码如下所示：

```c
static bool joydev_match(struct input_handler *handler, struct input_dev *dev)
{
    /* Disable blacklisted devices */
    if (joydev_dev_is_blacklisted(dev))
        return false;

    /* Avoid absolute mice */
    if (joydev_dev_is_absolute_mouse(dev))
        return false;

    return true;
}

static bool joydev_dev_is_blacklisted(struct input_dev *dev)
{
    const struct input_device_id *id;

    for (id = joydev_blacklist; id->flags; id++) {
        if (input_match_device_id(dev, id)) {
            dev_dbg(&dev->dev,
                "joydev: blacklisting '%s'\n", dev->name);
            return true;
        }
    }

    return false;
}

static const struct input_device_id joydev_blacklist[] = {
    /* Avoid touchpads and touchscreens */
    {
        .flags = INPUT_DEVICE_ID_MATCH_EVBIT |
                INPUT_DEVICE_ID_MATCH_KEYBIT,
        .evbit = { BIT_MASK(EV_KEY) },
        .keybit = { [BIT_WORD(BTN_TOUCH)] = BIT_MASK(BTN_TOUCH) },
    },
    /* Avoid tablets, digitisers and similar devices */
    {
        .flags = INPUT_DEVICE_ID_MATCH_EVBIT |
                INPUT_DEVICE_ID_MATCH_KEYBIT,
        .evbit = { BIT_MASK(EV_KEY) },
        .keybit = { [BIT_WORD(BTN_DIGI)] = BIT_MASK(BTN_DIGI) },
    },
    /* Disable accelerometers on composite devices */
    ACCEL_DEV(USB_VENDOR_ID_SONY, USB_DEVICE_ID_SONY_PS3_CONTROLLER),
    ACCEL_DEV(USB_VENDOR_ID_SONY, USB_DEVICE_ID_SONY_PS4_CONTROLLER),
    ACCEL_DEV(USB_VENDOR_ID_SONY, USB_DEVICE_ID_SONY_PS4_CONTROLLER_2),
    ACCEL_DEV(USB_VENDOR_ID_SONY, USB_DEVICE_ID_SONY_PS4_CONTROLLER_DONGLE),
    ACCEL_DEV(USB_VENDOR_ID_THQ, USB_DEVICE_ID_THQ_PS3_UDRAW),
    ACCEL_DEV(USB_VENDOR_ID_NINTENDO, USB_DEVICE_ID_NINTENDO_PROCON),
    ACCEL_DEV(USB_VENDOR_ID_NINTENDO, USB_DEVICE_ID_NINTENDO_CHRGGRIP),
    ACCEL_DEV(USB_VENDOR_ID_NINTENDO, USB_DEVICE_ID_NINTENDO_JOYCONL),
    ACCEL_DEV(USB_VENDOR_ID_NINTENDO, USB_DEVICE_ID_NINTENDO_JOYCONR),
    { /* sentinel */ }
};
```

从函数的名称中就能得知该函数的作用，再执行`joydev_connect`函数前会先运行`joydev_match`函数，`match`函数一般可以用来过滤或者对不同的设备进行不通的处理。

在`joydev.c`驱动的代码定义了一个黑名单列表，而`match`函数的作用是用来对这些黑名单中的设备进行过滤，`Nintendo`手柄就正好在这个黑名单中，不仅有`Nintendo`手柄，还是索尼的 PS 手柄也在黑名单中。

至于为什么`Nintendo`手柄会在 Linux 手柄驱动的黑名单中无从得知，只能从代码的注释中猜测一二：一般手柄会带有加速度传感器，用来玩一些支持体感类的游戏，比如健身环，可能`Nintendo`手柄的加速度传感器的功能在 Linux 驱动中还未实现，从`joydev_event`可以看出，Linux 的手柄驱动仅支持坐标系和按键功能，所以把支持加速度传感器的手柄给禁用了。

# 3 总结

到本篇文章结束，关于 USB 游戏手柄部分的研究就结束了，接下来就是研究其他 USB 设备，经过了 USB 游戏手柄的一番折腾，对 USB HID 驱动还有 input 驱动都有了一定的了解，对后续的研究也能有非常大的助力。

- - -
