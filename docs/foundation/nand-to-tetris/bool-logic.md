# 布尔逻辑
每种数字设备——比如个人电脑、手机或者网络路由器—都是基于一组用于储存和处理信息的芯片构建而成的。虽然这些芯片在外形上、构成上不尽相同，但是它们都是由相同的构造模块制造而成的：基本的逻辑门 （logic gates）。这些逻辑门可能会由很多不同的原材料和制造工艺来实现，但是其逻辑行为在所有的计算机中都是一致的。

在本章中，我们从原始的逻辑门“与非门”出发，以此来构建其他逻辑门。得到的是一组相当标准的逻辑门，这些逻辑门在后面将会被用来构建计算机的处理和存储芯片。

## 背景知识
### 布尔代数
布尔代数处理布尔型（也称二进制型）数值，比较典型的有 true/false、1/0、yes/no、on/off 等等。在这里我们使用1 和0。

布尔型函数（Boolean function）是指输入输出数值均为布尔型数值的函数。因为计算机硬件基于二进制数据的表示和处理，所以布尔函数在硬件体系结构的描述、构建和优化过程中扮演着十分重要的角色。那么正确地表达和分析布尔函数是迈向构建计算机体系结构的第一步。

两个二进制变量x和y可以构成的布尔函数共有16种。这是因为每个变量可以取0或1，所以两个变量组合起来可以形成4种不同的输入(00, 01, 10, 11)。对于每种输入，函数的输出可以是0或1，因此对于每种输入都有两种可能的输出。其中最基础的有 AND、OR、NOT、NAND、NOR 等函数。

Nand 函数以及 Nor 函数在理论上还有个有趣的特征：And/Or/Not 算子都可以只用Nand 或 Nor 函数来建构，（比如，x Ory=（x Nand x） Nand （y Nand y））。

既然每个布尔函数都能够通过规范表示法由 And、Or 和 Not 构成，那么每个布尔函数也能仅使用 Nand函数来构成，即所有的 16 种布尔函数都可以通过 Nand 构成。

这个结论有比较深远的实际意义：一旦在物理上实现了 Nand 功能，就可以使用很多这样的物理设备，通过特定的连接方式来构建任何布尔函数的硬件实现。

这也是《从 Nand 到俄罗斯方块》书名的由来，Nand 就是最底层的逻辑门。

### 门逻辑
门（gate）是用来实现布尔函数的物理设备，它的内部结构就是门的逻辑。

好比复杂的布尔函数能够通过相对简单的函数来表达一样，复杂的门电路也是由很多基本的门组成的。最简单的门是由微小的开关设备（称为晶体管，transistors）构成，这些微小开关设备按照设计的拓扑结构进行连接，来实现整个门的功能。

计算机开发者不需要关心门电路物理上的细节与内部结构，原始的门电路（primitive gate）能够被看成黑箱子，我们可以应用这些黑箱子来实现逻辑操作；至于黑箱子内部的实现，我们并不关心。硬件设计人员从这些基本的门开始，将它们进行连接，来实现功能更复杂的复合（composite）门电路。

### 硬件描述语言（HDL）
今天，硬件设计者再也不会用他们的那双手来制造硬件了。取而代之的是，他们在计算机工作站上设计和优化芯片结构，使用结构化的建模形式比如硬件描述语言（Hardroare
Description Language），或HDL。

设计者通过编写 HDL 程序来描述芯片的结构，该程序将接受严格的测试。该测试是虚拟的，使用计算机进行仿真的虚拟测试：有个特殊的软件工具称为硬件仿真器（hardware
simulator），它接受输入的HDL 程序，在计算机内存中构建该虚拟芯片的内存映像。然后，设计者就可以让仿真器来测试芯片，通过输入变量的不同组合产生仿真的芯片的输出。将
这些仿真的输出结果与预期的值进行比较，来验证我们的设计是否正确，是否满足客户的要求。

## 规范详述
这一章介绍了一些典型的门电路

- Nand 门
- 基本逻辑门：Not,Aand,Or,Xor,Multiplexor,Demultiplexor
- 多位基本门：多位 Not,多位 And,多位 Or,多位 Multiplexor
- 多通道逻辑门：多通道 Or,多通道 Multiplexor,多通道 Demultiplexor

## 项目
### Not 

```
CHIP Not {
  IN in;
  OUT out;
  PARTS:
  Nand(a = in, b = in, out = out);
}
```

### And
```
CHIP And {
    IN a, b;
    OUT out;
    
    PARTS:
    Nand(a = a, b = b, out = o1);
    Nand(a = a, b = b, out = o2);
    Nand(a = o1, b = o2, out = out);
}
```

### Or
```
CHIP Or {
    IN a, b;
    OUT out;

    PARTS:
    Not(in = a, out = o1);
    Not(in = b, out = o2);
    Nand(a = o1, b = o2, out = out);
}
```

### Xor
```
CHIP Xor {
    IN a, b;
    OUT out;

    PARTS:
    Not(in = b, out = n0);
    And(a = a, b = n0, out = a0);
    Not(in = a, out = n1);
    And(a = n1, b = b, out = a1);
    Or(a = a0, b = a1, out = out);
}
```

### Mux
```
/** 
 * Multiplexor:
 * if (sel = 0) out = a, else out = b
 */

CHIP Mux {
    IN a, b, sel;
    OUT out;

    PARTS:
    And(a = b, b = sel, out = o1);

    Not(in = sel, out = notSel);
    And(a = a, b = notSel, out = o2);

    Or(a = o1, b = o2, out = out);
}
```

### DMux
```
/**
 * Demultiplexor:
 * [a, b] = [in, 0] if sel = 0
 *          [0, in] if sel = 1
 */
CHIP DMux {
    IN in, sel;
    OUT a, b;

    PARTS:
    Not(in = sel, out = n1);
    And(a = in, b = n1, out = a);
    And(a = in, b = sel, out = b);
}
```

### Not16
```
CHIP Not16 {
    IN in[16];
    OUT out[16];

    PARTS:
    Not(in = in[0], out = out[0]);
    Not(in = in[1], out = out[1]);
    Not(in = in[2], out = out[2]);
    Not(in = in[3], out = out[3]);
    Not(in = in[4], out = out[4]);
    Not(in = in[5], out = out[5]);
    Not(in = in[6], out = out[6]);
    Not(in = in[7], out = out[7]);
    Not(in = in[8], out = out[8]);
    Not(in = in[9], out = out[9]);
    Not(in = in[10], out = out[10]);
    Not(in = in[11], out = out[11]);
    Not(in = in[12], out = out[12]);
    Not(in = in[13], out = out[13]);
    Not(in = in[14], out = out[14]);
    Not(in = in[15], out = out[15]);
}
```

### And16
```
CHIP And16 {
    IN a[16], b[16];
    OUT out[16];

    PARTS:
    And(a = a[0], b = b[0], out = out[0]);
    And(a = a[1], b = b[1], out = out[1]);
    And(a = a[2], b = b[2], out = out[2]);
    And(a = a[3], b = b[3], out = out[3]);
    And(a = a[4], b = b[4], out = out[4]);
    And(a = a[5], b = b[5], out = out[5]);
    And(a = a[6], b = b[6], out = out[6]);
    And(a = a[7], b = b[7], out = out[7]);
    And(a = a[8], b = b[8], out = out[8]);
    And(a = a[9], b = b[9], out = out[9]);
    And(a = a[10], b = b[10], out = out[10]);
    And(a = a[11], b = b[11], out = out[11]);
    And(a = a[12], b = b[12], out = out[12]);
    And(a = a[13], b = b[13], out = out[13]);
    And(a = a[14], b = b[14], out = out[14]);
    And(a = a[15], b = b[15], out = out[15]);
}
```

### Or16
```
CHIP Or16 {
    IN a[16], b[16];
    OUT out[16];

    PARTS:
    Or(a = a[0], b = b[0], out = out[0]);
    Or(a = a[1], b = b[1], out = out[1]);
    Or(a = a[2], b = b[2], out = out[2]);
    Or(a = a[3], b = b[3], out = out[3]);
    Or(a = a[4], b = b[4], out = out[4]);
    Or(a = a[5], b = b[5], out = out[5]);
    Or(a = a[6], b = b[6], out = out[6]);
    Or(a = a[7], b = b[7], out = out[7]);
    Or(a = a[8], b = b[8], out = out[8]);
    Or(a = a[9], b = b[9], out = out[9]);
    Or(a = a[10], b = b[10], out = out[10]);
    Or(a = a[11], b = b[11], out = out[11]);
    Or(a = a[12], b = b[12], out = out[12]);
    Or(a = a[13], b = b[13], out = out[13]);
    Or(a = a[14], b = b[14], out = out[14]);
    Or(a = a[15], b = b[15], out = out[15]);
}
```

### Mux16
```
/**
 * 16-bit multiplexor: 
 * for i = 0, ..., 15:
 * if (sel = 0) out[i] = a[i], else out[i] = b[i]
 */
CHIP Mux16 {
    IN a[16], b[16], sel;
    OUT out[16];

    PARTS:
    Mux(a = a[0], b = b[0], sel = sel, out = out[0]);
    Mux(a = a[1], b = b[1], sel = sel, out = out[1]);
    Mux(a = a[2], b = b[2], sel = sel, out = out[2]);
    Mux(a = a[3], b = b[3], sel = sel, out = out[3]);
    Mux(a = a[4], b = b[4], sel = sel, out = out[4]);
    Mux(a = a[5], b = b[5], sel = sel, out = out[5]);
    Mux(a = a[6], b = b[6], sel = sel, out = out[6]);
    Mux(a = a[7], b = b[7], sel = sel, out = out[7]);
    Mux(a = a[8], b = b[8], sel = sel, out = out[8]);
    Mux(a = a[9], b = b[9], sel = sel, out = out[9]);
    Mux(a = a[10], b = b[10], sel = sel, out = out[10]);
    Mux(a = a[11], b = b[11], sel = sel, out = out[11]);
    Mux(a = a[12], b = b[12], sel = sel, out = out[12]);
    Mux(a = a[13], b = b[13], sel = sel, out = out[13]);
    Mux(a = a[14], b = b[14], sel = sel, out = out[14]);
    Mux(a = a[15], b = b[15], sel = sel, out = out[15]);
}
```

### Or8Way
```
CHIP Or8Way {
    IN in[8];
    OUT out;

    PARTS:
    Or(a = in[0], b = in[1], out = o1);
    Or(a = o1, b = in[2], out = o2);
    Or(a = o2, b = in[3], out = o3);
    Or(a = o3, b = in[4], out = o4);
    Or(a = o4, b = in[5], out = o5);
    Or(a = o5, b = in[6], out = o6);
    Or(a = o6, b = in[7], out = out);
}
```

### Mux4Way16
```
/**
 * 4-way 16-bit multiplexor:
 * out = a if sel = 00
 *       b if sel = 01
 *       c if sel = 10
 *       d if sel = 11
 */
CHIP Mux4Way16 {
    IN a[16], b[16], c[16], d[16], sel[2];
    OUT out[16];
    
    PARTS:
    Mux16(a = a, b = b, sel = sel[0], out = o1);
    Mux16(a = c, b = d, sel = sel[0], out = o2);

    Mux16(a = o1, b = o2, sel = sel[1], out = out);
}
```

### Mux8Way16
```
/**
 * 8-way 16-bit multiplexor:
 * out = a if sel = 000
 *       b if sel = 001
 *       c if sel = 010
 *       d if sel = 011
 *       e if sel = 100
 *       f if sel = 101
 *       g if sel = 110
 *       h if sel = 111
 */
CHIP Mux8Way16 {
    IN a[16], b[16], c[16], d[16],
       e[16], f[16], g[16], h[16],
       sel[3];
    OUT out[16];

    PARTS:
    Mux4Way16(a = a, b = b, c = c, d = d, sel = sel[0..1], out = o1);
    Mux4Way16(a = e, b = f, c = g, d = h, sel = sel[0..1], out = o2);

    Mux16(a = o1, b = o2, sel = sel[2], out = out);
}
```

### DMux4Way
```
/**
 * 4-way demultiplexor:
 * [a, b, c, d] = [in, 0, 0, 0] if sel = 00
 *                [0, in, 0, 0] if sel = 01
 *                [0, 0, in, 0] if sel = 10
 *                [0, 0, 0, in] if sel = 11
 */
CHIP DMux4Way {
    IN in, sel[2];
    OUT a, b, c, d;

    PARTS:
    DMux(in = in, sel = sel[1], a = o1, b = o2);

    DMux(in = o1, sel = sel[0], a = a, b = b);
    DMux(in = o2, sel = sel[0], a = c, b = d);
}
```

### DMux8Way
```
/**
 * 8-way demultiplexor:
 * [a, b, c, d, e, f, g, h] = [in, 0,  0,  0,  0,  0,  0,  0] if sel = 000
 *                            [0, in,  0,  0,  0,  0,  0,  0] if sel = 001
 *                            [0,  0, in,  0,  0,  0,  0,  0] if sel = 010
 *                            [0,  0,  0, in,  0,  0,  0,  0] if sel = 011
 *                            [0,  0,  0,  0, in,  0,  0,  0] if sel = 100
 *                            [0,  0,  0,  0,  0, in,  0,  0] if sel = 101
 *                            [0,  0,  0,  0,  0,  0, in,  0] if sel = 110
 *                            [0,  0,  0,  0,  0,  0,  0, in] if sel = 111
 */
CHIP DMux8Way {
    IN in, sel[3];
    OUT a, b, c, d, e, f, g, h;

    PARTS:
    DMux4Way(in = in, sel = sel[1..2], a = o1, b = o2, c = o3, d = o4);
    DMux(in = o1, sel = sel[0], a = a, b = b);
    DMux(in = o2, sel = sel[0], a = c, b = d);
    DMux(in = o3, sel = sel[0], a = e, b = f);
    DMux(in = o4, sel = sel[0], a = g, b = h);
}
```