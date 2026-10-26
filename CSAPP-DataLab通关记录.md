# CSAPP Data Lab 通关记录：从位运算到 IEEE 754


## 发布信息

|字段|内容|
|-|-|
|标题|CSAPP Data Lab 通关记录：从位运算到 IEEE 754|
|分类|后端 / C++|
|标签|CSAPP、Data Lab、位运算、IEEE 754、C 语言、系统能力|
|摘要|在运算符极度受限的前提下完成 CSAPP Data Lab 的 13 个函数：记录德摩根律、掩码扩散、补码取负等位运算套路，IEEE 754 浮点的三个实现方向，以及踩过的 7 个坑。|
|封面|`./btest` 全绿截图（待补）|

\---

## 1\. 这个 lab 在做什么

背景：我 C++ 语法和数据结构还算熟练（刷过几百道 LeetCode），但计算机组成原理主要靠看视频，对位运算和底层的位级表示一直没底。Data Lab 要求“只用极其有限的运算符”实现 13 个函数，覆盖位运算、补码和 IEEE 754 浮点——正好戳在这块空白上。

|题|分值|考点|
|-|-|-|
|`bitXor`|1|禁用 `\|` 实现异或|
|`tmin`|1|补码最小值|
|`isTmax`|1|溢出与边界|
|`allOddBits`|2|掩码构造|
|`negate`|2|补码取负|
|`isAsciiDigit`|3|区间判断与符号位|
|`conditional`|3|用位运算实现 `x ? y : z`|
|`isLessOrEqual`|3|比较与溢出|
|`logicalNeg`|4|不用 `!` 判断是否为 0|
|`howManyBits`|4|二分 + 掩码|
|`floatScale2`|4|浮点 ×2|
|`floatFloat2Int`|4|浮点 → 整数|
|`floatPower2`|4|整数指数 → 浮点位模式|

总耗时：从 `bitXor` 到 `floatPower2` 大约 4 小时；最难的一题是 `floatFloat2Int`。

\---

## 2\. 环境与工作流

环境：Windows + WSL2（Ubuntu 22.04）+ gcc 11.4.0。

```bash
# 32 位编译支持（lab 的 Makefile 用了 -m32）
sudo apt install -y gcc-multilib libc6-dev-i386

cd \~/cs/datalab-handout
make              # 编译 btest
./btest           # 跑全部（一开始 0/36 是正常的）
./btest -f bitXor # 只跑一题
./dlc bits.c      # 检查有没有用禁用运算符、超操作数上限
```

核心循环（这就是全部工作流）：

```
读注释里的题面 → 实现功能 → 写 return → make → ./btest -f 函数名 → 看红色报错 → 改
```

注意：`./btest` 的报错格式是 `Gives X, Should be Y`，它不给思路，只给反例——这恰好是最好的反馈。

\---

## 3\. 位运算题的 5 个套路

这 13 题看着杂，其实就 5 招。

### ① 德摩根律：把禁用运算符换掉

德摩根律的口诀是“否定进括号，符号翻转，与或互换”：

```
一般形式：
\~(a | b) = \~a \& \~b
\~(a \& b) = \~a | \~b

变形（把“或”换成被禁用的形式）：
a | b = \~(\~a \& \~b)
```

`bitXor` 禁用 `|`，但异或需要“或”：

```c
int bitXor(int x, int y) {
  return \~(\~x \& \~y) \& \~(x \& y);   // (x|y) \& \~(x\&y)
}
```

### ② 掩码扩散：从 8 位拼出 32 位

常量只能用 0\~255，所以 `0xAAAAAAAA` 要自己拼（浮点题不受这条限制）：

```c
int mask = 0xAA;          // 0x000000AA
mask |= mask << 8;        // 0x0000AAAA
mask |= mask << 16;       // 0xAAAAAAAA
```

`allOddBits` 判断“奇数位是否全 1”就是 `(x \& mask) == mask`。

### ③ 补码：取反加一就是取负

补码的权重是 `-2³¹ + 2³⁰ + … + 2⁰`，**最高位的权重是负的**：

```c
int negate(int x) { return \~x + 1; }
```

它的一个高级用法是：**`x | -x` 的符号位只有在 `x == 0` 时才是 0**，于是：

```c
int logicalNeg(int x) { return ((x | (\~x + 1)) >> 31) + 1; }
```

### ④ 符号位：`>> 31` 得到 0 或 -1

`isAsciiDigit` 判断 `0x30 ≤ x ≤ 0x39`，做两个减法看符号位：

```c
int lower = (x + \~0x30 + 1) >> 31;   // x - 0x30 < 0 ?
int upper = (0x39 + \~x + 1) >> 31;   // 0x39 - x < 0 ?
return !lower \& !upper;
```

### ⑤ `!` 与 `!!`：相等判断和归一化

* `!(a ^ b)` 表示 **a 等于 b**（异或为 0）；
* `!!x` 把“非零”归一成 1，把 0 归一成 0。

`isTmax` 就靠这两招：

```c
int isTmax(int x) {
  int t = x + 1;            // Tmax → Tmin，-1 → 0
  return !(\~t ^ x) \& !!t;   // \~t == x 且 t != 0
}
```

> 严格来说，`Tmax + 1` 在有符号整数里是溢出（C 标准里是 UB），这里依赖 gcc 的补码回绕行为——lab 环境默认允许这么写，但面试时最好主动提一句，反而加分。

### ⑥ 一个让我真正看懂补码的瞬间

研究补码的形式之后我发现：`Tmax = 0111…1`，加 1 溢出就变成 `1000…0 = Tmin`，而 Tmin 取反又正好等于 Tmax，也就是 **`Tmax + 1 = \~Tmax`**。这条式子让我第一次“看见”了补码：它不只是“取反加一”的公式，而是一个首尾相接的环。

也正是在这一步，我意识到常量 `1` 在这个 lab 里有多重要：受运算符限制，几乎每道题都要先做一次等效变换才能下手，而“加一”“取反”“判断是否为 1”就是最常用的三种变形。做完这 13 题，我对底层位运算的理解比看视频时扎实得多。

\---

## 4\. 三个浮点题 + 一张 IEEE 754 图（全文重点）

### 4.1 单精度浮点长什么样

```
 31  30          23 22                          0
+----+--------------+-----------------------------+
| S  |      Exp     |          Fraction           |
+----+--------------+-----------------------------+
  1         8                      23
```

* 第 31 位是符号位；
* 第 23\~30 位是 8 位阶码，最大 `11111111 = 255`，但这个值是保留的（Inf / NaN），正常规格化数只用到 254；
* 第 0\~22 位是尾数（小数点后的部分）。

规格化数的值：

```
(-1)^S × 1.F × 2^(Exp - 127)
```

* **Exp 是移码**：存的是 `真实指数 + 127`，`Exp = 127` 对应 2⁰；
* **尾数有隐藏的 1**：`1.F` 中的 “1.” 不存储，白赚一位精度。

四类数：

|类型|Exp|Fraction|值|
|-|-|-|-|
|规格化|1 \~ 254|任意|`±1.F × 2^(Exp-127)`|
|非规格化|0|≠ 0|`±0.F × 2^-126`|
|零|0|0|`±0`|
|无穷|255|0|`±∞`|
|NaN|255|≠ 0|非数|

### 4.2 三个“为什么”

**为什么用移码？** 让指数可以按无符号数直接比较，硬件不用处理符号；于是正浮点数的位模式大小关系 = 数值大小关系。

**为什么隐藏 1？** 规格化后最高位永远是 1，不存也不丢信息，于是有 24 位有效数字（23 + 1）。

**为什么要有非规格化数？** 否则 0 和最小规格化数 2⁻¹²⁶ 之间有空档，精度会断崖式下跌。`0.F × 2⁻¹²⁶` 把这个区间平滑填满，叫**渐进下溢**。

### 4.3 记住这几个数

|量|值|
|-|-|
|最大规格化数|`(2-2⁻²³) × 2¹²⁷ ≈ 3.4 × 10³⁸`|
|最小规格化数|`2⁻¹²⁶ ≈ 1.18 × 10⁻³⁸`|
|最小非规格化数|`2⁻¹⁴⁹ ≈ 1.4 × 10⁻⁴⁵`|
|有效位数|24 位 ≈ 7 位十进制|
|能精确表示的最大整数|`2²⁴ = 16777216`|

### 4.4 floatScale2：位模式 ×2

核心：**阶码 +1 就是 ×2**；非规格化数整体左移；溢出变无穷。

```c
unsigned floatScale2(unsigned uf) {
  unsigned sign = uf \& 0x80000000;
  unsigned exp  = (uf >> 23) \& 0xFF;
  if (exp == 0xFF) return uf;                  // Inf / NaN
  if (exp == 0)    return sign | (uf << 1);    // 非规格化：整体左移
  exp++;
  if (exp == 0xFF) return sign | 0x7F800000;   // 溢出 → 无穷
  return sign | (exp << 23) | (uf \& 0x007FFFFF);
}
```

> 这里有个容易被忽略的点：`uf << 1` 能同时处理“×2”和“进位到阶码”——尾数最高位溢出时自然进位，正好过渡到最小规格化数。

### 4.5 floatFloat2Int：浮点 → 整数

核心：**用阶码决定移多少位**。先补上隐藏的 1 得到 24 位有效数，再按 `E = Exp - 127` 左移或右移。

```c
int floatFloat2Int(unsigned uf) {
  int sign = (uf >> 31) \& 1;
  int exp  = (uf >> 23) \& 0xFF;
  int frac = uf \& 0x7FFFFF;
  int E = exp - 127;
  if (exp == 0xFF) return 0x80000000;   // NaN / Inf
  if (exp == 0 || E < 0) return 0;      // 非规格化 或 |f| < 1
  if (E > 30) return 0x80000000;        // 溢出
  unsigned sig = frac | 0x800000;       // 补上隐藏的 1
  unsigned val = (E >= 23) ? (sig << (E - 23)) : (sig >> (23 - E));
  return sign ? -(int)val : (int)val;   // 负数向 0 截断
}
```

（附录 A 里是我最终提交的版本，变量名和注释略有不同，逻辑一致。）

### 4.6 floatPower2：指数 → 位模式

核心：分三段构造——上溢给无穷、下溢给 0、正常区间按 `Exp = x + 127` 拼；非规格化区间把 1 放到尾数对应位。

```c
unsigned floatPower2(int x) {
  if (x > 127)  return 0x7F800000;          // 2^128 溢出 → +∞
  if (x < -149) return 0;                   // 小于最小非规格化数
  if (x >= -126) return (unsigned)(x + 127) << 23;  // 规格化
  return 1u << (x + 149);                   // 非规格化：尾数 = 2^(x+149)
}
```

**三个题其实是同一件事的三种方向**：拆位 → 理解语义 → 重组。

\---

## 5\. 我踩过的坑

|#|坑|现象|原因|以后怎么防|
|-|-|-|-|-|
|1|`if (exp = 0xFF)`|函数永远返回 `uf`|赋值当比较，条件恒真|看 gcc 的 `-Wparentheses` 警告|
|2|`\&\&` 写成 `\&`|阶码变成 0/1|逻辑与 vs 按位与|取位一律用 `\&`|
|3|漏分号 ×2|编译失败|手快|先打 `;` 再填表达式|
|4|变量名写成 `a/b`|`undeclared`|思路照搬没改参数名|抄思路也要重命名|
|5|`isTmax` 写了个“空转”|恒返回 0|`\~(!y) + y` 在 y≠0 时恒等于 x，等于什么都没做|**先代数化简，再堆中间变量**|Tmax+1==Tmin==~Tmax，但-1+1==~（-1），我想将-1排除，用了~(!(x+1))+y，但是是空转，没有排除-1。。。
|6|`isAsciiDigit` 边界差 1|`0x39` 被判错|`>>31` 只能区分“<0”和“≥0”，`x-0x39 == 0` 时会被误判成越界|手测 4 个边界值：`0x2F/0x30/0x39/0x3A`|
|7|`make` 报 `bits/libc-header-start.h`|编译失败|缺 32 位库|`apt install gcc-multilib libc6-dev-i386`|

符号打错，=当成==等不必多说，在浮点数的习题上我最开始并不理解其表示为什么是符号位加阶码加尾数，所以关于exp的东西屡屡出错，在自己写了几个浮点数及其二进制表示后我开始了解这个表示及其精妙之处，关于位运算，二进制表示我的见解是写几个具体数，自己表示，自己运算再和熟悉的十进制联想，这样能更深刻的理解和运算（只是个人意见）
\---

## 6\. 做完之后我能回答的面试题

1. **`0.1 + 0.2` 为什么 ≠ `0.3`？** 0.1 的二进制是无限循环小数，尾数只能截断到 23 位，两个带误差的数相加，结果自然不是精确的 0.3。
2. **为什么要有非规格化数？** 填满 0 和最小规格化数之间的空档（渐进下溢），并表示 0。
3. **阶码为什么用移码？** 让正浮点数可以按无符号整数比较，硬件比较器更简单。
4. **float 能精确表示多大的整数？** 2²⁴ = 16777216，超过后只有部分整数能精确表示。
5. **NaN 怎么判断？** 用 `isnan()`；`NaN != NaN` 恒为真。
6. **`-0.0` 和 `0.0` 相等吗？** `==` 为真，但 `1.0 / -0.0 = -∞`。
7. **float 和 double 的区别？** 1+8+23 vs 1+11+52，double 约 15\~16 位十进制有效数字。
8. **浮点数能直接用 `==` 比较吗？** 不能，用 `fabs(a-b) < eps`。

> 在写完datalab的题目后，我对位运算有了更加深刻的了解，对与，或，非，浮点数的二进制表示更加理解，在今后的学习和工作中能更容易理解溢出，浮点数的计算和表示，关于位运算逻辑和边界情况也会更加注意

\---

## 附录 A：完整代码

（只贴了 13 个函数本身，`bits.c` 顶部的 `#include` 和注释块省略。）

```c
int bitXor(int x, int y) {
  return \~(\~x \& \~y) \& \~(x \& y);
} // x^y = (x|y) \& \~(x\&y)；x|y 用德摩根律展开成 \~(\~x \& \~y)

int tmin(void) {
  return 1 << 31;
} // int 是 32 位，补码最高位权重是 -2^31，所以 1<<31 就是最小值

int isTmax(int x) {
  int t = x + 1;
  return !(\~t ^ x) \& !!t;
} // Tmax+1 == Tmin，Tmin 取反 == Tmax；-1+1 == 0，所以还要排除 -1

int allOddBits(int x) {
  int mask = 0xAA;
  mask = mask | (mask << 8);
  mask = mask | (mask << 16);
  return !((x \& mask) ^ mask);
} // 0xA = 1010，移位拼出 0xAAAAAAAA；取奇数位后与掩码比较

int negate(int x) {
  return \~x + 1;
} // -x == \~x + 1

int isAsciiDigit(int x) {
  int lowerBound = (x + \~0x30 + 1) >> 31;   // x - 0x30 < 0 ?
  int upperBound = (0x39 + \~x + 1) >> 31;   // 0x39 - x < 0 ?
  return !lowerBound \& !upperBound;
} // 补码右移 31 位得到符号位（0 或 -1），! 把非 0 归一成 0

int conditional(int x, int y, int z) {
  int mask = \~(!!x) + 1;
  return (mask \& y) | (\~mask \& z);
} // mask 取 0 或 -1：x != 0 时 mask = -1，按位与 y；x == 0 时 mask = 0，\~mask = -1，按位与 z

int isLessOrEqual(int x, int y) {
  int signx = (x >> 31) \& 1;
  int signy = (y >> 31) \& 1;
  int diffsign = signx ^ signy;             // 符号位不同为 1，相同为 0
  int diff = y + \~x + 1;
  int signdiff = (diff >> 31) \& 1;
  int case1 = diffsign \& signx;             // 异号且 x 为负
  int case2 = !diffsign \& !signdiff;        // 同号且 y - x >= 0
  return case1 | case2;
} // 整数比较要防溢出，先看符号位：异号时 x 为负即 x <= y；同号时 y-x 的符号位决定

int logicalNeg(int x) {
  return ((x | (\~x + 1)) >> 31) + 1;
} // x | (-x) 的符号位：x == 0 时是 0，否则是 1；>>31 展开成 0 / -1，再 +1

int howManyBits(int x) {
  int sign, mask, op;
  sign = x >> 31;
  x = x ^ sign;                 // 负数取反，变成非负
  op = !!(x >> 16) << 4;        // 二分：先看高 16 位有没有 1
  x >>= op;
  mask = !!(x >> 8) << 3;
  x >>= mask;
  op += mask;
  mask = !!(x >> 4) << 2;
  x >>= mask;
  op += mask;
  mask = !!(x >> 2) << 1;
  x >>= mask;
  op += mask;
  mask = !!(x >> 1) << 0;
  x >>= mask;
  op += mask;
  return op + x + 1;            // 还要 +1 位符号位
}

unsigned floatScale2(unsigned uf) {
  unsigned sign = uf \& 0x80000000;
  unsigned exp  = (uf >> 23) \& 0xFF;

  if (exp == 0xFF) return uf;                     // Inf / NaN：原样返回
  if (exp == 0)    return sign | (uf << 1);       // 非规格化：整体左移
  exp++;                                          // 正常数：阶码 +1 = ×2
  if (exp == 0xFF) return sign | 0x7F800000;      // 溢出：变成无穷
  return sign | (exp << 23) | (uf \& 0x007FFFFF);
}

int floatFloat2Int(unsigned uf) {
  int sign = (uf >> 31) \& 1;                     // 符号位
  int exp  = (uf >> 23) \& 0xFF;                  // 阶码
  int frac = uf \& 0x007FFFFF;                    // 尾数
  int E = exp - 127;                             // 实际指数（exp 有 127 的偏移量）
  int M = frac | 0x00800000;                     // 补上隐藏的 1

  // 1. 特殊值与溢出
  if (exp == 255 || E > 30) {
    return 0x80000000u;                          // NaN / Inf / 超出 int 范围
  }
  if (E < 0) {
    return 0;                                    // |f| < 1，截断后是 0
  }

  // 2. 按 E 移动尾数
  if (E >= 23) {
    M = M << (E - 23);                           // 左移，补全后面的 0
  } else {
    M = M >> (23 - E);                           // 右移，丢掉小数部分
  }

  // 3. 负数
  if (sign) {
    M = -M;                                      // 转成补码形式
  }
  return M;
}

unsigned floatPower2(int x) {
  // 上溢：大于最大规格化数的指数 127
  if (x > 127) {
    return 0x7F800000;                           // 返回正无穷 +INF
  }

  // 下溢：小于非规格化数能表示的最小指数 -149
  if (x < -149) {
    return 0;                                    // 返回 0
  }

  // 规格化数范围：-126 <= x <= 127
  if (x >= -126) {
    int exp = x + 127;                           // 阶码 Exp = x + 127
    return exp << 23;                            // 符号位和尾数都是 0
  }

  // 非规格化数范围：-149 <= x < -126，用尾数表示 2^x
  return 1u << (x + 149);                        // 尾数 = 2^(x+149)
}
```

\---

## 附录 B：参考链接

* CSAPP 官方 lab 页：[https://csapp.cs.cmu.edu/3e/labs.html](https://csapp.cs.cmu.edu/3e/labs.html)
* Data Lab writeup：[https://csapp.cs.cmu.edu/3e/datalab.pdf](https://csapp.cs.cmu.edu/3e/datalab.pdf)

\---