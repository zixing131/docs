

# 浅析 JavaScript Obfuscator（一） - 先知社区

浅析 JavaScript Obfuscator（一）

- - -

原来的账号找不回了，尴尬，新开一个吧。

最近看到一个用`JavaScript Obfuscator`做前端 JS 代码混淆的案例，由于临近年关坐等放假，就闲得蛋疼简单分析了一下。

## JavaScript Obfuscator Tool

首先，我们在[官网](https://obfuscator.io/)看一下，`JavaScript Obfuscator`提供了众多的选项作为混淆入参：

-   基础
    
    -   Disable Console Output（禁用控制台输出）
        
        禁用控制台全局调用所有脚本，默认为`false`
        
    -   Self Defending（自我防御）
        
        使混淆代码能够抵抗格式化和变量重命名，例如 JavaScript 代码美化工具，默认为`false`
        
    -   Debug Protection（调试保护）
        
        使浏览器开发者工具中的调试器无法正常使用，默认为`false`
        
-   字符串变换
    
    -   String Array（字符串数组）
        
        用特殊的字符串数组替换原有的字符串值，默认为`true`
        
    -   String Array Rotate（字符串数组轮换）
        
        使字符串数组进行位移，默认为`true`
        
    -   String Array Shuffle（字符串数组乱序）
        
        随机打乱字符串数组，默认为`true`
        
    -   String Array Threshold（字符串数组阈值）
        
        指定字符串值放入字符串数组的比例，默认为`0.8`
        
    -   String Array Index Shift（字符串数组索引位移）
        
        对字符串数组调用的索引进行位移，默认为`true`
        
    -   String Array Indexes Type（字符串数组索引类型）
        
        指定字符串数组调用的索引类型，默认为`Hexadecimal Number`
        
    -   String Array Calls Transform（字符串数组调用变换）
        
        使字符串数组调用取值的过程变得更加复杂困难，默认为`false`，默认阈值为`0.5`
        
    -   String Array Wrappers Count（字符串数组包装器数量）
        
        指定字符串数组包装器的数量，默认为`1`
        
    -   String Array Wrappers Type（字符串数组包装器类型）
        
        指定字符串数组包装器的类型，默认为`variable`
        
    -   String Array Wrappers Chained Calls（字符串数组包装器链式调用）
        
        启用字符串数组包装器的链式调用，默认为`true`
        
    -   String Array Encoding（字符串数组编码）
        
        指定字符串数组的编码方式，默认为`无`
        
    -   Split Strings（分割字符串）
        
        将字符串拆分为指定长度的块，默认为`false`，默认块长度为`10`
        
-   标识符变换
    
    -   Identifier Names Generator（标识符名称生成器）
        
        指定标识符名称的生成方式，默认为`Hexadecimal`
        
    -   Identifiers Prefix（标识符前缀）
        
        指定全局标识符名称的前缀，默认为`无`
        
    -   Rename Globals（重命名全局变量）
        
        使用声明来混淆全局变量和函数名称，默认为`false`
        
    -   Rename Properties（重命名属性）
        
        重命名属性名称，默认为`false`
        
-   其他变换
    
    -   Compact（紧凑）
        
        使代码变为一行，默认为`true`
        
    -   Simplify（简化）
        
        通过简化对额外的代码进行混淆，默认为`true`
        
    -   Transform Object Keys（变换对象键）
        
        使对象的键进行变换，默认为`false`
        
    -   Numbers To Expressions（数字转表达式）
        
        使数字转换为表达式，默认为`false`
        
    -   Control Flow Flattening（平坦化控制流）
        
        使代码控制流扁平化，默认为`false`，默认阈值为`0.75`
        
    -   Dead Code Injection（死代码注入）
        
        在代码中插入随机的死代码块，默认为`false`
        

官方也贴心的给出了四种预设模式，分别为：Default（默认，高性能）、Low（低混淆，高性能）、Medium（中等混淆，最佳性能）、High（高混淆、低性能），那本文就从默认开始分析。

同时，我选择`Crypto-JS`项目中的`md5.js`作为示例，它即有丰富的变量属性函数定义，也有多样的算数计算过程，应该可以体现出混淆后的大部分特点。

## 默认级别混淆特点

由于默认级别混淆的预设参数中只启用了`String Array`相关的几个基础项、`Compact`，以及`Simplify`，因此通过对比分析可以发现，源代码的逻辑几乎没有太多的变化，只是呈现出几个明显的特点。

### 特点一：十六进制化

首先，最直观最典型的肯定是混淆后的代码中充斥着大量的十六进制或是类似十六进制的表示。

举个例子：

```plain
for (var i = 0; i < 16; i++) {
    var offset_i = offset + i;
    var M_offset_i = M[offset_i];
    M[offset_i] = (
        (((M_offset_i << 8)  | (M_offset_i >>> 24)) & 0x00ff00ff) |
        (((M_offset_i << 24) | (M_offset_i >>> 8))  & 0xff00ff00)
    );
}
```

它被混淆后，代码如下：

```plain
for (var _0x546cf3 = 0x0; _0x546cf3 < 0x10; _0x546cf3++) {
    var _0x54e4d4 = _0x1effab + _0x546cf3,
        _0x323639 = _0x5e5b11[_0x54e4d4];
    _0x5e5b11[_0x54e4d4] = 
        (_0x323639 << 0x8 | _0x323639 >>> 0x18) & 0xff00ff |
        (_0x323639 << 0x18 | _0x323639 >>> 0x8) & 0xff00ff00;
}
```

所有十进制数都被转换成了十六进制，就连变量名等标识符都替换成了以`_0x`开头的伪十六进制表示。

因此，即使逻辑原封不动，但从视觉上就造成了一定的阅读障碍，代码长度的增加和反计算习惯的数值表示会让代码量较少的读者本能的觉得变得很复杂。

### 特点二：字符串化

其次，它将字符串类型的值都使用字符串数组的形式进行了替换。

且属性也由原来`c.m`的方式改为了`c['m']`，函数调用从`c.f()`改为了`c['f']()`，使得它们都可以借助字符串进行一定程度的改写。

举个例子：

```plain
var MD5 = C_algo.MD5 = Hasher.extend({
    _doReset: function () {
        this._hash = new WordArray.init([
            0x67452301, 0xefcdab89,
            0x98badcfe, 0x10325476
        ]);
    },
    ...
});
```

它被混淆后，代码如下：

```plain
var _0x227d20 = _0xc3f48e['MD5'] = _0x38037[_0x509931(0x1c9)]({
    '_doReset': function() {
        var _0x148e27 = _0x509931;
        this[_0x148e27(0x1bb)] = new _0x522fc1[(_0x148e27(0x1c1))]([
            0x67452301, 0xefcdab89,
            0x98badcfe, 0x10325476
        ]);
    },
});
```

原来的`C_algo.MD5`变成了`_0xc3f48e['MD5']`，`Hasher.extend()`变成了`_0x38037[_0x509931(0x1c9)]()`，其他类似。

其中，代表`Hasher`对象的`extend`函数的字符串被表示成了`_0x509931(0x1c9)`，这很可能就是字符串数组的特定取值函数，后面我们再进行详细分析。

而混淆后的代码中还是可以看见明文字符串，这是因为默认级别中`String Array Threshold`参数的值仅为`0.8`，也就是只会转换 80% 的字符串值。

### 特点三：逻辑简化

逻辑简化在默认级别出现的场景并不多，主要是`else`语句块和`return`语句。

举个例子：

```plain
if (typeof exports === "object") {
    module.exports = exports = factory(require("./core"));
}
else if (typeof define === "function" && define.amd) {
    define(["./core"], factory);
}
else {
    factory(root.CryptoJS);
}
```

它被混淆后，代码如下：

```plain
if (typeof exports === _0x4be726(0x1bf)) module[_0x4be726(0x1b7)] = exports = _0x3138cf(require(_0x4be726(0x1bc)));
else typeof define === _0x4be726(0x1ca) && define[_0x4be726(0x1af)] ? define([_0x4be726(0x1bc)], _0x3138cf) : _0x3138cf(_0x527611[_0x4be726(0x1b3)]);
```

很明显，原来的`if-else if-else`被简写成了`if-else`，`else`块中使用三元表达式来覆盖原条件逻辑。

而另一种，则是利用 JavaScript 中一些语言特性进行了某种意义上的简化。

举个例子：

```plain
function (CryptoJS) {
    ...
    return CryptoJS.MD5;
}
```

它被混淆后，代码如下：

```plain
function (CryptoJS) {
    return function(_0x14f7a1) {
        ...
    }(Math), _0x43590a[_0x4e10a0(0x1cb)];
}
```

当 JavaScript 的`return`语句中包含了多个返回值时，实际只会返回最后一个。

因此，它将函数中的部分逻辑放在`return`的第一个返回值内，并包装成了一个立即执行的匿名函数，继续提高阅读难度。

## 字符串数组取值

从分析出的特点可以看出，『特点一』和『特点三』其实只需要我们转换一下思维习惯就可以很轻松的适应，甚至不需要太多逆向。

而『特点二』中的字符串数组以及它的取值过程，则是接下来重点照顾的对象，由三个主要函数构成。

首先，是它的字符串数组定义函数，没什么弯弯绕绕，仅仅多了个闭包而已，很简单：

```plain
function _0x10e7() {
    var _0x4bbd32 = ['object', '2133236AnmzGt', 'init', '37782620StZnsn', 'floor', '2284455sDMAIQ', '_data', '18EBGUjf', 'sin', 'clone', 'extend', 'function', 'MD5', 'words', '899535gIwEuD', '_process', 'length', 'abs', 'call', '10rXFioH', '_createHmacHelper', 'amd', 'algo', '1037863FtBCAS', 'WordArray', 'CryptoJS', '4QLUdFg', 'sigBytes', 'Hasher', 'exports', 'lib', '_createHelper', 'HmacMD5', '_hash', './core', '1428792YZRUwn', '2351988oecqJc'];
    _0x10e7 = function() {
        return _0x4bbd32;
    };
    return _0x10e7();
}
```

只是数组中多了`2133236AnmzGt`、`37782620StZnsn`等一些奇怪的项，到底有什么用呢？继续往下看。

有了数组，自然就需要对它进行取值。但是大家应该还记得默认级别的参数中，有一个`String Array Index Shift`会在字符串数组调用时对索引进行一定量的偏移。

既然有偏移那肯定就得复位，所以第二个就是对数组索引复位并取值的函数：

```plain
function _0x3126(_0x30a9d6, _0x49b9ba) {
    var _0x10e724 = _0x10e7();
    return _0x3126 = function(_0x3126ad, _0x475b54) {
        _0x3126ad = _0x3126ad - 0x1aa;
        var _0x45a84 = _0x10e724[_0x3126ad];
        return _0x45a84;
    }, _0x3126(_0x30a9d6, _0x49b9ba);
};
```

应该也很容易理解，表示数组的变量是`_0x10e724`，索引的随机偏移量是`0x1aa`。

假设字符串数组取值到这就完成了，我们来试试能不能正常获取到想要的值。

以之前示例中提到的代表`Hasher`对象的`extend`函数的字符串的取值`_0x509931(0x1c9)`为例，`_0x509931`即为取值函数`_0x3126`，`0x1c9`复位后得到`31`，回到字符串数组中找到第 32 项，发现得到的值是`_createHelper`。

取值结果不对，那说明肯定还有一个操作把字符串数组项顺序给打乱了，这也对应了默认级别参数中的`String Array Rotate`和`String Array Shuffle`。

因此，找到最后一个函数，一个立即执行的匿名函数，用来对字符串数组进行『洗牌』：

```plain
(function(_0x2226ad, _0x1b76a1) {
    var _0x38ef3e = _0x3126,
        _0x1a727d = _0x2226ad();
    while (!![]) {
        try {
            var _0x11c77c = -parseInt(_0x38ef3e(0x1b1)) / 0x1 + -parseInt(_0x38ef3e(0x1c0)) / 0x2 + parseInt(_0x38ef3e(0x1c4)) / 0x3 * (-parseInt(_0x38ef3e(0x1b4)) / 0x4) + parseInt(_0x38ef3e(0x1ad)) / 0x5 * (-parseInt(_0x38ef3e(0x1be)) / 0x6) + parseInt(_0x38ef3e(0x1cd)) / 0x7 + parseInt(_0x38ef3e(0x1bd)) / 0x8 * (parseInt(_0x38ef3e(0x1c6)) / 0x9) + parseInt(_0x38ef3e(0x1c2)) / 0xa;
            if (_0x11c77c === _0x1b76a1) break;
            else _0x1a727d['push'](_0x1a727d['shift']());
        } catch (_0x4fda4d) {
            _0x1a727d['push'](_0x1a727d['shift']());
        }
    }
}(_0x10e7, 0x95e73));
```

首先，看到`push`和`shift`，可以快速推断出它其实就是不停的把数组中的首项移到末项，做一个简单的数组位移操作。

但是什么时候才会停止呢？这里面有一个很关键的判断条件`_0x11c77c === _0x1b76a1`。

也就是说，需要将取到的多个指定索引中的字符串值转换成整数后，通过各种数学运算得到一个特定的数。

我们都知道，JavaScript 中有另一个语言特性，当需要将一个包含非数字字符的字符串转换成数字时，它会取开头的数字字符，直至遇到非数字字符，如`'123abc'`会被转换成`123`。

还记得之前发现的`2133236AnmzGt`那几个奇怪的值么？它们其实就是在混淆时被随机插入的特定字符串，用来作为数组位移操作停止的条件项。

这也使得这个函数的运行结果无法轻易被肉眼观察或简单口算出来，需要通过浏览器控制台等方式调试或执行一次才能获取到最终的字符串数组：

```plain
['length', 'abs', 'call', '10rXFioH', '_createHmacHelper', 'amd', 'algo', '1037863FtBCAS', 'WordArray', 'CryptoJS', '4QLUdFg', 'sigBytes', 'Hasher', 'exports', 'lib', '_createHelper', 'HmacMD5', '_hash', './core', '1428792YZRUwn', '2351988oecqJc', 'object', '2133236AnmzGt', 'init', '37782620StZnsn', 'floor', '2284455sDMAIQ', '_data', '18EBGUjf', 'sin', 'clone', 'extend', 'function', 'MD5', 'words', '899535gIwEuD', '_process']
```

再看这个数组的第 32 项，就是我们想要的`extend`，Bingo！

至此，我们就可以将混淆后代码中所有的字符串数组调用全部替换成目标字符串，基本不会再过多的影响代码的阅读和理解了。

## 参考

1.  [JavaScript Obfuscator Tool (v2.9.0)](https://obfuscator.io/)
