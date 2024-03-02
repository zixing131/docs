
# USB 设备开发：从入门到实践指南（二）

[↓↓↓](javascript:)  
  
  
  
[↑↑↑](javascript:)

48 分钟之前  
[安全工具&安全开发](https://paper.seebug.org/category/tools/) · [经验心得](https://paper.seebug.org/category/experience/) · [404 专栏](https://paper.seebug.org/category/404team/)

## 目录

-   [1 模拟鼠标](#1)
-   [2 模拟 USB 游戏手柄](#2-usb)
-   [3 动态修改 Ubuntu 驱动](#3-ubuntu)
-   [4 本篇总结](#4)
-   [5 参考链接](#5)

**作者：Hcamael@知道创宇 404 实验室  
时间：2024 年 2 月 28 日**

接着上一篇遗留的问题，继续对 USB 设备开发进行研究。

# 1 模拟鼠标

在[上一篇的 Paper](https://paper.seebug.org/3122/ "上一篇的Paper")中，我们尝试对 USB 键盘进行模拟，下一步再尝试对 USB 鼠标设备进行模拟。

在能成功模拟键盘的基础上，要实现鼠标的模拟是很容易的，只需在模拟键盘的`bash`脚本中以下修改两部分：

1.需要对协议进行修改，`1`表示的是键盘，`2`表示的是鼠标：

```bash
echo 2 > "${FUNCTIONS_DIR}/protocol" # mouse
```

2.需要对 HID 描述符进行修改，对于鼠标来说，至少需要两个按键（左右键）和 XY 二维坐标系（控制鼠标的移动）。但是现在的鼠标功能越来越多，比如在鼠标的中间还有一个滚轮，这就需要增加一个一维坐标系，和一个按键（中键）。另外，一般的游戏鼠标在大拇指位置还设置了两个用来翻页的按键。

根据上述描述的功能，来编写一个鼠标的 HID 描述符：

```plain
0x05, 0x01,          # USAGE_PAGE (Generic Desktop)
0x09, 0x02,          # USAGE (Mouse)
0xa1, 0x01,          # COLLECTION (Application)
0x85, 0x66,          # Report ID 0x66
0x09, 0x01,          # USAGE (Pointer)
0xa1, 0x00,          # COLLECTION (Physical)
0x05, 0x09,          # USAGE_PAGE (Button)
0x19, 0x01,          # USAGE_MINIMUM (0x1)
0x29, 0x05,          # USAGE_MAXIMUM (0x5)
0x15, 0x00,          # LOGICAL_MINIMUM (0)
0x25, 0x01,          # LOGICAL_MAXIMUM (1)
0x75, 0x01,          # REPORT_SIZE (1)
0x95, 0x05,          # REPORT_COUNT (5)
0x81, 0x02,          # INPUT (Data,Var,Abs)

0x95, 0x01,          # REPORT_COUNT (3)
0x75, 0x03,          # REPORT_SIZE (1)
0x81, 0x01,          # INPUT (Cnst,Array,Abs)

0x05, 0x01,          # USAGE_PAGE (Generic Desktop Controls)
0x09, 0x30,          # USAGE (X)
0x09, 0x31,          # USAGE (Y)
0x16, 0x01, 0x80,    # LOGICAL_MINIMUM (-0x7FFF)
0x26, 0xFF, 0x7F,    # LOGICAL_MAXIMUM (0x7FFF)
0x95, 0x02,          # REPORT_COUNT (2)
0x75, 0x10,          # REPORT_SIZE (16)
0x81, 0x06,          # INPUT (Data,Var,Rel)

0x09, 0x38,          # USAGE (Wheel)
0x15, 0x81,          # LOGICAL_MINIMUM (-0x7F)
0x25, 0x7F,          # LOGICAL_MAXIMUM (0x7F)
0x95, 0x01,          # REPORT_COUNT (1)
0x75, 0x08,          # REPORT_SIZE (8)
0x81, 0x06,          # INPUT (Data,Var,Rel)
0xc0,                # END_COLLECTION
0xc0                 # END_COLLECTION
```

在讲解上面的 HID 描述符的之前，首先需要了解一件事：鼠标的厂商如果想要生产一个 USB 免驱动的鼠标，并不是并不是任意增加按键就可以的，而是需要遵循 HID 规范文档，因为操作系统开发内置的鼠标驱动也是根据该文档进行开发的，除非厂商自行开发鼠标驱动代码并开源贡献给 Linux 内核，并且被采纳。

接下来讲解一下上面的 HID 描述符，首先定义了 Button 按键，一个按键占 1bit，1 表示按下，0 表示释放，一共有 5 个按键，共占了 5bit，由于需要 8bit 对齐，所以还有 3bit 的 padding。

这 5 个按键分别代表哪些按键呢？可以自己测试一下，或者搜索 Linux 鼠标驱动的源码，如下所示：

```c
// drivers/hid/usbhid/usbmouse.c
    input_report_key(dev, BTN_LEFT,   data[0] & 0x01); // 左键
    input_report_key(dev, BTN_RIGHT,  data[0] & 0x02); // 右键
    input_report_key(dev, BTN_MIDDLE, data[0] & 0x04); // 滚轮中键
    input_report_key(dev, BTN_SIDE,   data[0] & 0x08); // 侧边按键（翻页）
    input_report_key(dev, BTN_EXTRA,  data[0] & 0x10); // 侧边按键（翻页）
```

接着定义了一个 XY 直角坐标系，代表了鼠标是如何移动的，其值从 -0x7FFF 到 0x7FFF，一个轴占 2 字节，XY 轴共占 4 字节。

最后定义了滚轮，最小值为 -0x7F，最大值为 0x7F，共占 1 字节。

上述的三部分总共占据了 6 字节，因为定义了一个 USB 中断传输为 8 字节，所以还可以加上 2 字节的 padding，不添加也不会造成问题，内核不会因此而报错。

到此为止，一个鼠标我们就模拟成功了，按照上一篇的 Paper 中的步骤，运行`bash`脚本，就能通过写`/dev/hidg0`文件来控制主机上鼠标的点击与移动。

比如我们需要点击左键，代码如下所示：

```python
def sendBuf(buf):
    with open("/dev/hidg0", "wb") as f:
        f.write(bytes(buf))

mouse_buf = [0] * 8
mouse_buf[0] = 0b00000001
sendBuf(mouse_buf)
```

但是需要注意，跟键盘按键一样，按照上述代码操作后，表示的是按下左键，但是并不会释放，我们正常使用鼠标左键点击的实际过程其实是包含两部分的。首先，按下左键，然后释放左键。这才是一个完整的点击过程。如果我们要实现一个完整的点击过程，可以参考如下代码：

```python
def sendBuf(buf):
    with open("/dev/hidg0", "wb") as f:
        f.write(bytes(buf))

def clickLeft():
    mouse_buf = [0] * 8
    mouse_buf[0] |= 0b00000001
    sendBuf(mouse_buf)
    mouse_buf[0] &amp;= 0b11111110
    sendBuf(mouse_buf)
```

如果需要控制鼠标的其他功能，可以对上述代码进行简单的修改，比如定义的五个按键，只需要控制`mouse_buf[0]`的低 5bit 的值。控制鼠标横向移动，则是控制`mouse_buf[1:3]`的值，纵向移动则是`mouse_buf[3:5]`，控制滚轮则是`mouse_buf[5]`。具体细节大家可以私下自行测试研究。

# 2 模拟 USB 游戏手柄

第二部分就是本篇的核心内容：模拟一个 USB 游戏手柄。

该部分内容说简单也简单，在能成功模拟 USB 鼠标键盘之后，也可以很容易的模拟出一个 USB 游戏手柄。但是如果深入研究当今常用的游戏手柄，将会遇到很多问题。

首先，我们从简单的学起，先实现一个普通的游戏手柄，首先看我们要如何修改模拟鼠标键盘的脚本，如下所示：

```bash
    echo 0 > "${FUNCTIONS_DIR}/protocol" # None
    echo 0 > "${FUNCTIONS_DIR}/subclass" # No subclass

    echo 0x045e > idVendor
    echo 0x028e > idProduct
```

当 protocol 等于 0 的时候，不会被鼠标键盘的驱动识别到，而游戏手柄的驱动会根据 idVendor/idProduct 匹配到该 USB 设备，在 Linux 上，手柄的驱动的代码一般位于`drivers/input/joystick/xpad.c`，可以查看该驱动中能识别的部分 XBox 游戏手柄如下所示：

```c
xpad_device[] = {
......
    { 0x045e, 0x0202, "Microsoft X-Box pad v1 (US)", 0, XTYPE_XBOX },
    { 0x045e, 0x0285, "Microsoft X-Box pad (Japan)", 0, XTYPE_XBOX },
    { 0x045e, 0x0287, "Microsoft Xbox Controller S", 0, XTYPE_XBOX },
    { 0x045e, 0x0288, "Microsoft Xbox Controller S v2", 0, XTYPE_XBOX },
    { 0x045e, 0x0289, "Microsoft X-Box pad v2 (US)", 0, XTYPE_XBOX },
    { 0x045e, 0x028e, "Microsoft X-Box 360 pad", 0, XTYPE_XBOX360 },
    { 0x045e, 0x028f, "Microsoft X-Box 360 pad v2", 0, XTYPE_XBOX360 },
    { 0x045e, 0x0291, "Xbox 360 Wireless Receiver (XBOX)", MAP_DPAD_TO_BUTTONS, XTYPE_XBOX360W },
    { 0x045e, 0x02d1, "Microsoft X-Box One pad", 0, XTYPE_XBOXONE },
    { 0x045e, 0x02dd, "Microsoft X-Box One pad (Firmware 2015)", 0, XTYPE_XBOXONE },
    { 0x045e, 0x02e3, "Microsoft X-Box One Elite pad", MAP_PADDLES, XTYPE_XBOXONE },
    { 0x045e, 0x0b00, "Microsoft X-Box One Elite 2 pad", MAP_PADDLES, XTYPE_XBOXONE },
    { 0x045e, 0x02ea, "Microsoft X-Box One S pad", 0, XTYPE_XBOXONE },
    { 0x045e, 0x0719, "Xbox 360 Wireless Receiver", MAP_DPAD_TO_BUTTONS, XTYPE_XBOX360W },
    { 0x045e, 0x0b0a, "Microsoft X-Box Adaptive Controller", MAP_PROFILE_BUTTON, XTYPE_XBOXONE },
    { 0x045e, 0x0b12, "Microsoft Xbox Series S|X Controller", MAP_SELECT_BUTTON, XTYPE_XBOXONE },
......
```

从上面的代码可以看出，我们模拟的手柄名称为：`Microsoft X-Box 360 pad`。

接下来还需要编写 HID 描述符，如下所示：

```plain
0x05, 0x01,        // Usage Page (Generic Desktop Ctrls)
0x09, 0x05,        // Usage (Game Pad)
0xA1, 0x01,        // Collection (Application)
0x15, 0x00,        //   Logical Minimum (0)
0x25, 0x01,        //   Logical Maximum (1)
0x35, 0x00,        //   Physical Minimum (0)
0x45, 0x01,        //   Physical Maximum (1)
0x75, 0x01,        //   Report Size (1)
0x95, 0x0E,        //   Report Count (14)
0x05, 0x09,        //   Usage Page (Button)
0x19, 0x01,        //   Usage Minimum (0x01)
0x29, 0x0E,        //   Usage Maximum (0x0E)
0x81, 0x02,        //   Input (Data,Var,Abs,No Wrap,Linear,Preferred State,No Null Position)

0x95, 0x02,        //   Report Count (2)
0x81, 0x01,        //   Input (Const,Array,Abs,No Wrap,Linear,Preferred State,No Null Position)

0x05, 0x01,        //   Usage Page (Generic Desktop Ctrls)
0x25, 0x07,        //   Logical Maximum (7)
0x46, 0x3B, 0x01,  //   Physical Maximum (315)
0x75, 0x04,        //   Report Size (4)
0x95, 0x01,        //   Report Count (1)
0x65, 0x14,        //   Unit (System: English Rotation, Length: Centimeter)
0x09, 0x39,        //   Usage (Hat switch)
0x81, 0x42,        //   Input (Data,Var,Abs,No Wrap,Linear,Preferred State,Null State)

0x65, 0x00,        //   Unit (None)
0x95, 0x01,        //   Report Count (1)
0x81, 0x01,        //   Input (Const,Array,Abs,No Wrap,Linear,Preferred State,No Null Position)

0x26, 0xFF, 0x00,  //   Logical Maximum (255)
0x46, 0xFF, 0x00,  //   Physical Maximum (255)
0x09, 0x30,        //   Usage (X)
0x09, 0x31,        //   Usage (Y)
0x09, 0x32,        //   Usage (Z)
0x09, 0x35,        //   Usage (Rz)
0x75, 0x08,        //   Report Size (8)
0x95, 0x04,        //   Report Count (4)
0x81, 0x02,        //   Input (Data,Var,Abs,No Wrap,Linear,Preferred State,No Null Position)

0x75, 0x08,        //   Report Size (8)
0x95, 0x01,        //   Report Count (1)
0x81, 0x01,        //   Input (Const,Array,Abs,No Wrap,Linear,Preferred State,No Null Position)

0xC0,              // End Collection
```

分析一下上面的 HID 描述符：

1.  定义了 14 个按键，占 14bit，8bit 对其，有 2bit 的 padding 填充。
2.  定义了 Hat Switch，占 4bit，分别代表了上下左右，按住按键移动的距离单位用 cm（厘米），移动的最大值是 315cm。
3.  接下来就是摇杆控制的 XY 还有体感控制的 Z 和 Rz，每个方向的值占 8bit，也就是 1 字节，4 个方向共占了 4 字节，单位没有再次单独定义，所以也是用 cm，最大值为 255。
4.  最后还有 1 字节的 padding 填充，所有按键定义总共加起来有 8 字节。

所有环节准备就绪了（这次 USB 主机设备选择的是 Linux 主机），接下来就可以运行脚本，然后可以在 Linux 主机上看到以下信息：

```plain
$ sudo dmesg
[91788.951749] usb 3-2: new high-speed USB device number 6 using xhci_hcd
[91789.279411] usb 3-2: New USB device found, idVendor=045e, idProduct=028e, bcdDevice= 2.00
[91789.279438] usb 3-2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[91789.279442] usb 3-2: Product: My Test Pro Controller
[91789.279447] usb 3-2: Manufacturer: test Nintendo Switch Pro
[91789.279452] usb 3-2: SerialNumber: 0202020001314
[91789.285488] input: test Nintendo Switch Pro My Test Pro Controller as /devices/pci0000:00/0000:00:16.0/0000:0b:00.0/usb3/3-2/3-2:1.0/0003:045E:028E.0003/input/input8
[91789.286937] hid-generic 0003:045E:028E.0003: input,hidraw1: USB HID v1.01 Gamepad [test Nintendo Switch Pro My Test Pro Controller] on usb-0000:0b:00.0-2/input0
# 注意事项：别看日志中显示的是 Switch 手柄，因为这些字符串都可以自行随意设置，而内核驱动是根据 idVendor/idProduct 来设备手柄是什么设备的，所以对 Linux 内核来说这就是一个 XBox 手柄。
```

接着我们发现在`/dev/input`目录下多了两个文件，如下所示：

```shell
$ ls -alF /dev/input/by-id
total 0
drwxr-xr-x 2 root root 120 Jan 25 20:44 ./
drwxr-xr-x 4 root root 360 Jan 25 20:44 ../
lrwxrwxrwx 1 root root   9 Jan 25 20:44 usb-test_Nintendo_Switch_Pro_My_Test_Pro_Controller_0202020001314-event-joystick -> ../event6
lrwxrwxrwx 1 root root   6 Jan 25 20:44 usb-test_Nintendo_Switch_Pro_My_Test_Pro_Controller_0202020001314-joystick -> ../js1
```

接着我们可以通过直接读`/dev/input/js1`文件：`cat /dev/input/js1|hexdump -C`来查看手柄的输入，测试过程代码如下所示：

```plain
# 树莓派 USB 设备端
def send(buf):
    with open("/dev/hidg0", "wb") as f:
        f.write(bytes(buf))
buf = [0] * 8
buf[0] = 0b00000001
send(buf)

# Linux 主机端
cat /dev/input/js1|hexdump -C
00000000  70 7f 79 05 00 00 81 00  70 7f 79 05 00 00 81 01  |p.y.....p.y.....|
00000010  70 7f 79 05 00 00 81 02  70 7f 79 05 00 00 81 03  |p.y.....p.y.....|
00000020  70 7f 79 05 00 00 81 04  70 7f 79 05 00 00 81 05  |p.y.....p.y.....|
00000030  70 7f 79 05 00 00 81 06  70 7f 79 05 00 00 81 07  |p.y.....p.y.....|
00000040  70 7f 79 05 00 00 81 08  70 7f 79 05 00 00 81 09  |p.y.....p.y.....|
00000050  70 7f 79 05 00 00 81 0a  70 7f 79 05 00 00 81 0b  |p.y.....p.y.....|
00000060  70 7f 79 05 00 00 81 0c  70 7f 79 05 00 00 81 0d  |p.y.....p.y.....|
00000070  70 7f 79 05 01 80 82 00  70 7f 79 05 01 80 82 01  |p.y.....p.y.....|
00000080  70 7f 79 05 01 80 82 02  70 7f 79 05 01 80 82 03  |p.y.....p.y.....|
00000090  70 7f 79 05 00 00 82 04  70 7f 79 05 01 80 82 05  |p.y.....p.y.....|


000000a0  90 12 7a 05 01 00 01 00  08 46 7a 05 01 00 01 01  |..z......Fz.....|
js1 文件的数据结构为：
struct js_event {
    __u32 time;     /* event timestamp in milliseconds */
    __s16 value;    /* value */
    __u8 type;      /* event type */
    __u8 number;    /* axis/button number */
};
```

我们可以写一个 python 脚本让 js1 的数据更有可视性，python 代码如下所示：

```python
#!/usr/bin/env python3
# -*- coding=utf-8 -*-

import os
import sys
import struct
import select

fmt_js0 = '<IhBB'
js0_size = struct.calcsize(fmt_js0)

# Open the joystick device.
fn_js0 = sys.argv[1]
js0dev = os.open(fn_js0, os.O_RDONLY)

# Create an epoll object
epoll = select.epoll()
epoll.register(js0dev, select.EPOLLIN)

old_js0 = None

while True:
    events = epoll.poll()
    for fd, event in events:
        if fd == js0dev:
            event_js0 = os.read(js0dev, js0_size)
            if event_js0 != old_js0:
                old_js0 = event_js0
                time, value, type, number = struct.unpack(fmt_js0, event_js0)
                print(f'JS0 - binary: {event_js0}')
                print('JS0 - time: {}, value: {}, type: {}, number: {}'.format(time, value, type, number))
```

下面展示一些脚本输出的数据样例，如下所示：

```plain
data = [0] * 8
buf[0] = 0b00000001
# 主机数据
JS0 - binary: b'\x00H\x81\x05\x01\x00\x01\x00'
JS0 - time: 92358656, value: 1, type: 1, number: 0

buf[0] = 0b00000000
JS0 - binary: b'LP\x82\x05\x00\x00\x01\x00'
JS0 - time: 92426316, value: 0, type: 1, number: 0

buf[0] = 0b10000000
JS0 - binary: b'(\xd2\x82\x05\x01\x00\x01\x07'
JS0 - time: 92459560, value: 1, type: 1, number: 7
```

从这我们可以看出，`type=1`应该表示为操作类型为操作 Button，除了按键还有滚轮方向键这些（XYZRz 和 Hat Switch），`number=x`表示第几个按键，按照我们编写的 HID 描述符来说，`number`的值应该在`0-13`之间，而`value`就是具体的值，对于按键来说，只能是 0 或者 1。

或者我们可以安装`joystick`包，使用命令：`sudo apt install joystick`，安装好以后，可以使用`jstest`命令来进行更加可视化的观察，如下所示：

```shell
$ jstest /dev/input/js1
Driver version is 2.1.0.
Joystick (test Nintendo Switch Pro My Test Pro Controller) has 6 axes (X, Y, Z, Rz, Hat0X, Hat0Y)
and 14 buttons (BtnA, BtnB, BtnC, BtnX, BtnY, BtnZ, BtnTL, BtnTR, BtnTL2, BtnTR2, BtnSelect, BtnStart, BtnMode, BtnThumbL).
Testing ... (interrupt to exit)
Axes:  0:-32767  1:-32767  2:-32767  3:-32767  4:     0  5:-32767 Buttons:  0:off  1:off  2:off  3:off  4:off  5:off  6:off  7:off   8:off  9:off 10:off 11:off 12:off 13:off
```

手柄的模拟第一阶段就到此为止，我们已经可以成功的模拟一个功能简单的游戏手柄。

接下来就是对游戏手柄的深入研究。（或者也不算是深入研究，只是笔者在研究游戏手柄最初阶段踩到的坑，从而多研究了一部分相关内容。）

在研究游戏手柄最初，考虑的方案是用现成的手柄连接到电脑上，然后抓包分析实际的手柄数据。

现成的手柄有两个，一个是`Switch Pro`，一个是国产手柄，该手柄可以伪装成`Switch Pro`手柄。

并且在 Linux 的源码中发现 Swithc 手柄的相关驱动：`drivers/hid/hid-nintendo.c`。

综合上述因素，选择了 Switch 手柄作为研究的切入点，但最终却发现选错了切入点。

正版的 Switch Pro 手柄，不管是接入 Windows 系统还是 Ubuntu 系统，都能抓到 USB 相关的数据包还有 HID 描述符，但是主机却无法正常使用手柄。在 Windows 上暂时没法研究，因为没有逆向 Windows 驱动的经验。不过经过一番搜索发现，`Switch Pro`手柄是不支持 USB 接入的，只能通过蓝牙接入主机。

那在 Ubuntu 上的 nintendo 驱动又是怎么回事呢？通过查看日志发现：

```plain
$ sudo dmesg
[93718.957323] usb 3-2: new full-speed USB device number 7 using xhci_hcd
[93719.300499] usb 3-2: New USB device found, idVendor=057e, idProduct=2009, bcdDevice= 2.00
[93719.300508] usb 3-2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[93719.300513] usb 3-2: Product: Pro Controller
[93719.300542] usb 3-2: Manufacturer: Nintendo Co., Ltd.
[93719.300555] usb 3-2: SerialNumber: 020002002001
[93719.315256] nintendo 0003:057E:2009.0004: hidraw1: USB HID v81.11 Joystick [Nintendo Co., Ltd. Pro Controller] on usb-0000:0b:00.0-2/input0
[93721.649661] nintendo 0003:057E:2009.0004: failed reading SPI flash; ret=-110
[93721.649674] nintendo 0003:057E:2009.0004: using factory cal for left stick
[93723.697308] nintendo 0003:057E:2009.0004: failed reading SPI flash; ret=-110
[93723.697321] nintendo 0003:057E:2009.0004: using factory cal for right stick
[93725.745498] nintendo 0003:057E:2009.0004: failed reading SPI flash; ret=-110
[93725.745509] nintendo 0003:057E:2009.0004: Failed to read left stick cal, using defaults; e=-110
[93727.793565] nintendo 0003:057E:2009.0004: failed reading SPI flash; ret=-110
[93727.793577] nintendo 0003:057E:2009.0004: Failed to read right stick cal, using defaults; e=-110
[93729.841197] nintendo 0003:057E:2009.0004: failed reading SPI flash; ret=-110
[93729.841208] nintendo 0003:057E:2009.0004: using factory cal for IMU
[93731.889086] nintendo 0003:057E:2009.0004: failed reading SPI flash; ret=-110
[93731.889099] nintendo 0003:057E:2009.0004: Failed to read IMU cal, using defaults; ret=-110
[93731.889105] nintendo 0003:057E:2009.0004: Unable to read IMU calibration data
[93733.936993] nintendo 0003:057E:2009.0004: Failed to set report mode; ret=-110
[93733.937724] nintendo 0003:057E:2009.0004: probe - fail = -110
[93733.937740] nintendo: probe of 0003:057E:2009.0004 failed with error -110
$ lsmod|grep hid
hid_nintendo           45056  0
ff_memless             24576  1 hid_nintendo
mac_hid                12288  0
hid_generic            12288  0
usbhid                 77824  0
hid                   180224  3 hid_nintendo,usbhid,hid_generic
```

发现 ubuntu 成功适配到了`hid_nintendo`驱动，但是却报了一堆的错误，猜测是这些错误导致手柄驱动注册失败，在 Linux 下能被正常识别的手柄应该像上面的案例一样，能在`/dev/input/`目录下生成`jsX`和`eventX`文件，因为在 Linux 上使用手柄的软件都是通过`/dev/input/jsX`文件来和手柄进行交互的。

但是目前的情况下，在`/dev/input`目录下并没有生成 Switch 手柄相关的文件。这个时候就是考虑对相关驱动进行研究调试，或者换设备了。

首先尝试最简单的方案，换国产手柄进行尝试，经过测试发现这个国产手柄问题更大。首先在 Windows 上，它会伪装成一个`Switch Pro`游戏手柄，但是前面说了，`Switch Pro`无法在 Windows 上正常使用，所以该手柄检测到无法正常使用时，会再次伪装成一个 XBox 手柄，这个时候就能被 Windows 正常识别和使用了。从这里看会感觉这个手柄还不错，优先伪装成 Swith，如果失败了再伪装成 XBox。

但是，该手柄在代码实现上估计有大 BUG，会导致`USBTree View`, `Wireshark`的`USBPcap`, 还有 Windows 的部分 USB 驱动崩溃（有可能是 USBPcap 导致的）。这个时候只能考虑重启电脑来修复问题，但是这个时候发现电脑无法正常关机，只能通过长按电源键强制关机。另外因为测试使用的 Linux 是装在 Windows 上的虚拟机，`vmware`在 Windows 上安装了一个 USB 驱动，来让主机接入虚拟机当中的，可能是同样的 BUG 导致在 Linux 系统上也无法正常识别到手柄相关驱动（就算是 nintendo 驱动也没识别到），并且不知是何原因，Windows 主机强制重启多次后，会导致 Linux 虚拟机图形界面的网卡管理出现问题，需要重新安装`network-manager`。

有一个猜测是：该手柄有防破解机制。

综上原因，我把该手柄丢进小黑屋，只用来打游戏，不拿来研究。

因此接下来只能考虑对相关驱动进行研究，如果只通过静态分析，难度很大，就算在有源码的情况下，纯静态分析也得花费很长一段时间。所以动态调试是一条必须要考虑的事情，但是由于现实因素，驱动位于内核中，调试驱动可以等于调试 Linux 内核，这不像调试一个普通的 ELF 程序，直接使用 gdb。调试内核需要有一个比较长的环境搭建的过程，本地并没有现成的环境。

由于动态调试的道路比较曲折，再考虑一下其他调试方案，有以下三种方案：

1.  使用 eBPF。
2.  驱动本身有调试输出，不过需要内核编译开启 DEBUG 参数。
3.  因为需要研究的是 Linux 驱动，而 Linux 的驱动大部分都是属于插件类型，可以在系统启动之后进行加载，卸载等动作，可以编辑驱动源码，添加调试输出，然后把原本的驱动卸载，加载修改后的驱动。

对于上述三个方案进行利弊分析：笔者虽然研究过 eBPF，但是研究的不深入，开发相关程序是比较花时间的。第二个方案，需要重新编译 Linux 内核，这个同样需要浪费大量时间。因此考虑了第三个方案，用时话费最小。

# 3 动态修改 Ubuntu 驱动

首先安装内核源码，代码如下所示：

```bash
# 需要包装安装的源码和当前内核同一版本
$ uname -r
6.5.0-14-generic
$ apt list linux-source-6.5.0 -a
Listing... Done
linux-source-6.5.0/jammy-updates,jammy-updates 6.5.0-15.15~22.04.1 all
linux-source-6.5.0/jammy-updates,jammy-updates 6.5.0-14.14~22.04.1 all
$ sudo apt install linux-source-6.5.0=6.5.0-14.14~22.04.1
# 移动到自己方便查看修改的目录
$ mkdir ~/kernelTest
$ mv /usr/src/linux-source-6.5.0.tar.bz2 ~/kernelTest
$ cd ~/kernelTest && tar xjf linux-source-6.5.0.tar.bz2 
```

接着编写了一个简单的脚本，当我们修改了相关驱动的时候，快速重新编译驱动然后加载进内核，脚本代码如下所示：

```shell
#!/bin/bash

DRIVER_NAME=$1
DRIVER_PATH="drivers/hid/hid-${DRIVER_NAME}.ko"

sudo rmmod $DRIVER_PATH

make CONFIG_HID_ROCCAT=n CONFIG_I2C_HID_CORE=n CONFIG_AMD_SFH_HID=n CONFIG_INTEL_ISH_HID=n -C /lib/modules/6.5.0-14-generic/build  M=/home/hehe/kernelTest/linux-source-6.5.0/drivers/hid/ modules -j8
./scripts/sign-file sha512 ./certs/signing_key.pem certs/signing_key.x509 $DRIVER_PATH

sudo insmod $DRIVER_PATH

tail -f /var/log/kern.log # 相当于dmesg
```

输出调试信息的方案有两种，如下所示：

```c
1. 参考了hid_dbg，重新定一个了
#define mhid_dbg(hid, fmt, ...) dev_printk(KERN_DEBUG, &(hid)->dev, fmt, ##__VA_ARGS__)
比如我可以把hid-nintendo.c中的hid_dbg都替换成mhid_dbg
2. 使用print_hex_dump函数，使用方法如下：
static int nintendo_hid_event(struct hid_device *hdev,
                  struct hid_report *report, u8 *raw_data, int size)
{
    struct joycon_ctlr *ctlr = hid_get_drvdata(hdev);

    if (size < 1)
        return -EINVAL;

    print_hex_dump(KERN_INFO, "JOYCON HID RECV: ", DUMP_PREFIX_OFFSET, 16, 1, raw_data, size, false);

    return joycon_ctlr_handle_event(ctlr, raw_data, size);
}
```

现在我们已经可以对 nintendo 驱动进行调试，笔者对调试过程进行了以下记录。

1.每个 hid 驱动首先关注`hid_driver`结构体，代码如下所示：

```c
static struct hid_driver nintendo_hid_driver = {
    .name       = "nintendo",
    .id_table   = nintendo_hid_devices,
    .probe      = nintendo_hid_probe,
    .remove     = nintendo_hid_remove,
    .raw_event  = nintendo_hid_event,

#ifdef CONFIG_PM
    .resume     = nintendo_hid_resume,
#endif
};
```

`.id_table`的结构数据表明匹配到当前驱动的条件，匹配成功后执行`.probe`函数，当接收到设备发来的数据时，触发`.raw_event`函数，设备移除时触发`.remove`函数，设备有可能会休眠，休眠结束后会触发`.resume`函数。

2.需要研究`nintendo_hid_probe`函数，经过研究发现`switch`手柄也有专门的协议，比如有定义握手包，有读取手柄芯片上数据的功能。然后发现一个很奇怪的事情，很大一块读取手柄数据的代码逻辑是为了判断该手柄是左手柄还是右手柄，玩过 switch 的同学就知道这代表了什么。

据我所知，switch 只有原装手柄才分左右手柄，然而左右手柄却不存在 usb 接口，`nintendo`驱动为什么要适配`USB_DEVICE_ID_NINTENDO_JOYCONL`和`USB_DEVICE_ID_NINTENDO_JOYCONR`呢？

并且通过抓包发现，测试的`Switch Pro`手柄并不会响应驱动中定义的协议，想到以前看到过研究 Switch 手柄的 Paper，里面展示过 Switch 手柄相关的协议流量，不过捕获的是`JOYCONL`和`JOYCONR`手柄的流量，但是正常情况下`Switch Pro`手柄也应该支持该协议，暂时不清楚原因。

通过逆向驱动中的 nintendo 协议，编写了以下脚本，当模拟`Switch Pro`设备成功后，监听`/dev/hidg0`，回应主机发来的请求，脚本代码如下所示：

```python
#!/usr/bin/env python3
# -*- coding=utf-8 -*-

import os
import sys
import select
import struct

DEV="/dev/hidg0"

'''
struct joycon_input_report {
    u8 id;
    u8 timer;
    u8 bat_con; /* battery and connection info */
    u8 button_status[3];
    u8 left_stick[3];
    u8 right_stick[3];
    u8 vibrator_report;

    union {
        struct joycon_subcmd_reply subcmd_reply;
        /* IMU input reports contain 3 samples */
        u8 imu_raw_bytes[sizeof(struct joycon_imu_data) * 3];
    };
} __packed;
struct joycon_subcmd_reply {
    u8 ack; /* MSB 1 for ACK, 0 for NACK */
    u8 id; /* id of requested subcmd */
    u8 data[]; /* will be at most 35 bytes */
} __packed;
'''
class JOYCON_INPUT_REPORT:
    MIN_SIZE = 0xD
    JC_INPUT_SUBCMD_REPLY = 0x21
    # def __init__(self, data):
    #     self.id, self.timer, self.bat_con, self.button_status, self.left_stick, self.right_stick, self.vibrator_report = struct.unpack('BBB3s3s3sB', data[:self.MIN_SIZE])
    #     self.subcmd_ack, self.subcmd_id = struct.unpack('BB', data[self.MIN_SIZE: self.MIN_SIZE+2])
    #     self.subcmd_data = data[self.MIN_SIZE+2:]
    def __init__(self):
        self.id = 0
        self.timer = 0
        self.batcon = 0
        self.button_status = b"\x00\x00\x00"
        self.left_stick = b"\x00\x00\x00"
        self.right_stick = b"\x00\x00\x00"
        self.vibrator_report = 0
        self.subcmd_ack = 1
        self.subcmd_id = 0
        self.subcmd_data = []

    def genPayload(self):
        payload = bytearray([self.id, self.timer, self.batcon])
        payload += self.button_status
        payload += self.left_stick
        payload += self.right_stick
        payload += bytearray([self.vibrator_report, self.subcmd_ack, self.subcmd_id])
        payload += bytearray(self.subcmd_data)
        return payload

'''
struct joycon_subcmd_request {
    u8 output_id; /* must be 0x01 for subcommand, 0x10 for rumble only */
    u8 packet_num; /* incremented every send */
    u8 rumble_data[8];
    u8 subcmd_id;
    u8 data[]; /* length depends on the subcommand */
} __packed;
'''
class JOYCON_SUBCMD_REQUEST:
    MIN_SIZE = 0xB
    JC_SUBCMD_REQ_DEV_INFO = 0x02
    JC_SUBCMD_SET_REPORT_MODE = 0x03
    JC_SUBCMD_SPI_FLASH_READ = 0x10
    JC_SUBCMD_SET_PLAYER_LIGHTS = 0x30
    JC_SUBCMD_ENABLE_IMU = 0x40
    JC_SUBCMD_ENABLE_VIBRATION = 0x48

    JC_CAL_USR_MAGIC_0 = 0xB2
    JC_CAL_USR_MAGIC_1 = 0xA1

    def __init__(self, data):
        self.output_id, self.packet_num, self.rumble_data, self.subcmd_id = struct.unpack('BB8sB', data[:self.MIN_SIZE])
        self.data = data[self.MIN_SIZE:]

    def parse(self):
        if self.subcmd_id == self.JC_SUBCMD_SPI_FLASH_READ:
            if len(self.data) == 5:
                report = JOYCON_INPUT_REPORT()
                report.subcmd_id = self.subcmd_id
                report.id = JOYCON_INPUT_REPORT.JC_INPUT_SUBCMD_REPLY
                report.subcmd_data = [0] * 5 + [self.JC_CAL_USR_MAGIC_0, self.JC_CAL_USR_MAGIC_1] + [0] * 27
                return report.genPayload()
        elif self.subcmd_id in [self.JC_SUBCMD_SET_REPORT_MODE, self.JC_SUBCMD_ENABLE_VIBRATION, self.JC_SUBCMD_ENABLE_IMU, self.JC_SUBCMD_SET_PLAYER_LIGHTS]:
            report = JOYCON_INPUT_REPORT()
            report.subcmd_id = self.subcmd_id
            report.id = JOYCON_INPUT_REPORT.JC_INPUT_SUBCMD_REPLY
            report.subcmd_data = [0] * 34
            return report.genPayload()
        elif self.subcmd_id == self.JC_SUBCMD_REQ_DEV_INFO:
            report = JOYCON_INPUT_REPORT()
            report.subcmd_id = self.subcmd_id
            report.id = JOYCON_INPUT_REPORT.JC_INPUT_SUBCMD_REPLY
            report.subcmd_data = [0] * 4 + [0x41, 0x42, 0x43, 0x44, 0x45, 0x46] + [0] * 24
            return report.genPayload()
        return None

class HIDParse(object):
    JC_OUTPUT_USB_CMD = 0x80
    JC_INPUT_USB_RESPONSE = 0x81

    def __call__(self, data):
        if len(data) == 2:
            if data[0] == self.JC_OUTPUT_USB_CMD:
                return bytearray([self.JC_INPUT_USB_RESPONSE, data[1]])
        elif len(data) >= JOYCON_SUBCMD_REQUEST.MIN_SIZE:
            subcmd_req = JOYCON_SUBCMD_REQUEST(data)
            return subcmd_req.parse()

        return None

class HIDSock:
    def __init__(self, dev):
        self.dev = dev
        self.parse = HIDParse()
        self.listenRead()

    def callback(self, data):
        print(f"[DEBUG]: do callback, data = {data}")
        info = self.parse(data)
        print(f"[DEBUG]: parse data: {info}")
        if info:
            with open(self.dev, "wb") as f:
                f.write(info)

    def listenRead(self):
        fd_read = open(self.dev, "rb")
        poll = select.poll()
        poll.register(fd_read, select.POLLIN)
        try:
            while True:
                events = poll.poll(1000)  # Timeout of 1000 ms
                for fd, event in events:
                    if event &amp; select.POLLIN:
                        data = os.read(fd, 64)
                        self.callback(data)
        finally:
            fd_read.close()

def sendBuf(buf):
    with open(DEV, "wb") as f:
        f.write(buf)

def main():
    hid = HIDSock(DEV)
    bufList = [
        0x21, 0x0D, 0x8E, 0x84, 0x00, 0x12, 0x01, 0x18, 0x80, 0x01, 0x18, 0x80, 0x80, 0x90, 0x10, 0x10, 
        0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00
    ]
    sendBuf(bytearray(bufList))

if __name__ == "__main__":
    main()
```

接着再看现在是否能成功加载 nintendo 驱动了，查看内核信息，如下所示：

```plain
$ sudo dmesg
Jan 25 22:35:37 hehe kernel: [98280.604727] usb 3-2: new high-speed USB device number 10 using xhci_hcd
Jan 25 22:35:37 hehe kernel: [98280.931988] usb 3-2: New USB device found, idVendor=057e, idProduct=2009, bcdDevice= 2.00
Jan 25 22:35:37 hehe kernel: [98280.932016] usb 3-2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
Jan 25 22:35:37 hehe kernel: [98280.932022] usb 3-2: Product: My Test Pro Controller
Jan 25 22:35:37 hehe kernel: [98280.932028] usb 3-2: Manufacturer: test Nintendo Switch Pro
Jan 25 22:35:37 hehe kernel: [98280.932034] usb 3-2: SerialNumber: 0202020001314
Jan 25 22:35:37 hehe kernel: [98280.939409] nintendo 0003:057E:2009.0007: hidraw1: USB HID v81.01 Joystick [test Nintendo Switch Pro My Test Pro Controller] on usb-0000:0b:00.0-2/input0
Jan 25 22:35:37 hehe kernel: [98281.006883] nintendo 0003:057E:2009.0007: using user cal for left stick
Jan 25 22:35:37 hehe kernel: [98281.008835] nintendo 0003:057E:2009.0007: using user cal for right stick
Jan 25 22:35:37 hehe kernel: [98281.010804] nintendo 0003:057E:2009.0007: Failed to read left stick cal, using defaults; e=-22
Jan 25 22:35:37 hehe kernel: [98281.012867] nintendo 0003:057E:2009.0007: Failed to read right stick cal, using defaults; e=-22
Jan 25 22:35:37 hehe kernel: [98281.014782] nintendo 0003:057E:2009.0007: using user cal for IMU
Jan 25 22:35:37 hehe kernel: [98281.024835] nintendo 0003:057E:2009.0007: controller MAC = 41:42:43:44:45:46
Jan 25 22:35:37 hehe kernel: [98281.028080] input: Nintendo Switch Pro Controller as /devices/pci0000:00/0000:00:16.0/0000:0b:00.0/usb3/3-2/3-2:1.0/0003:057E:2009.0007/input/input9
Jan 25 22:35:37 hehe kernel: [98281.028758] input: Nintendo Switch Pro Controller IMU as /devices/pci0000:00/0000:00:16.0/0000:0b:00.0/usb3/3-2/3-2:1.0/0003:057E:2009.0007/input/input10
$ ls -alF /dev/input/by-id
total 0
drwxr-xr-x 2 root root 100 Jan 25 22:35 ./
drwxr-xr-x 4 root root 360 Jan 25 22:35 ../
lrwxrwxrwx 1 root root   9 Jan 25 22:35 usb-test_Nintendo_Switch_Pro_My_Test_Pro_Controller_0202020001314-event-if00 -> ../event6
```

成功加载出`eventX`文件，但是却没加载出`jsX`文件，目前认为是`nintendo`驱动导致的问题。经过研究发现`hid-nintendo`驱动是最近几年才加入 Linux 内核的，也许该驱动代码还不完善，或者本身就没有考虑适配`Switch Pro`，目前没能想明白该驱动的实际用途。

不过也不能说该驱动毫无用处，如果我们想使用 Switch 手柄，仍然能通过读取`eventX`来获取手柄的输入，不过`eventX`的结构体和`jsX`的不同，`eventX`的结构体为`input_event`，结构体定义如下所示：

```c
struct input_event {
    struct timeval time;
    __u16 type;
    __u16 code;
    __s32 value;
};
```

目前还未对`input_event`结构体和`js_event`结构的关系进行进一步研究，猜测`jsX`的数据是根据`eventX`的数据进行一些处理后生成的，该问题待后续进一步研究。

# 4 本篇总结

通过本篇文章，我们了解了如何模拟一个 USB 鼠标，USB 游戏手柄设备，并且可以学习如何对 Linux 内核中的 HID 驱动进行修改然后输出相关调试信息。后续文章中，将会对`/dev/input/eventX`事件进行深入研究，还有会对非 HID 的 USB 进行研究学习。

# 5 参考链接

1.  [https://usb.org/sites/default/files/hut1\_4.pdf](https://usb.org/sites/default/files/hut1_4.pdf "https://usb.org/sites/default/files/hut1_4.pdf")
2.  [https://swf.com.tw/?p=1536#google\_vignette](https://swf.com.tw/?p=1536#google_vignette "https://swf.com.tw/?p=1536#google_vignette")

- - -
