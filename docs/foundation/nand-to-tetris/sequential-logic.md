# 时序逻辑
在前两章里面构建的所有的布尔芯片和算术芯片都是组合芯片（combinational chips）。组合芯片计算那些“输出结果仅依赖于其输入变量的排列组合”的函数。这些相关的简单
芯片提供很多重要的处理功能（像ALU一样），但是它们却不能维持自身的状态。

计算机不仅要能计算值，而且还需要存取数据，因而这些芯片必须配备记忆单元（memory elements）来保存数据，这些记忆单元是由时序芯片 （sequential chips） 组成。

记忆单元的实现是复杂的过程，涉及了同步、时钟和反馈回路。其中的大部分能够被封装到称为触发器（flip-flop）的底层时序门（sequential gate）中。本章将这些触发器作
为基预置模块来使用，详细介绍和构建所有典型的现代计算机所采用的记忆设备（memory devices），涉及的内容从二进制单元 （binary celis）到寄存器 （registers，注意：这里的寄存器不是 CPU 中的寄存器），再到存储块（memory banks）和计数器（counters）。

## 背景知识
### 时钟（Clock） 
在大多数计算机里，时间的流逝是用主时钟（master clock）来表示的，它提供连续的交变信号序列。其精确的硬件实现通常基于振荡器 （oscillator），其在两个信号值 0-1，或称“低电平-高电平（lowo-high，tick-tock）”之间交替变化。

两个相邻的上升沿之间的时间间隙称为时钟的周期（cycle），每个时钟周期模塑一个离散时间单元。通过使用硬件电路，这个信号同时被传送到计算机平台的每个时序芯片中。

### 触发器（Flip-Flops） 
计算机里最基本的时序单元是称为触发器的设备，它有几个种变体。在本书里我们使用称为数据触发器（Data Flip-Flop,DFF 或称 D触发器）的变体，其接口包含1比特位输入和1比特位输出。

另外，DFF 有个时钟输入，根据主时钟信号连续地交变。数据和时钟的输入使得 DFF 能够实现基于时间的行为 out（t）=in（t-1），这里 in 和 out是门的输入和输出值，t 是当前时钟周期。换句话说，DFF 简单地将前一个时间周期的输入值作为当前周期的输出。

由此可见，这种基本行为是所有计算机硬件维持自身状态的基础，从二进制单元到寄存器以至任意大的随机存取记忆单元（RAM）皆是如此。

### 寄存器（Registers） 
寄存是具有记忆功能的设备，它能够“储存”或称“记忆”某一时刻的值，实现经典的存储行为 out（t）=out（t—1）。

从另一个方面来说，DFF 仅能够输出它前一时钟周期的输入，也就是 out（t）=in（t—1）。这就告诉我们，可以通过 DFF 来实现寄存器，只需将后面的输出反馈到它的输入就可以了。

一旦实现了保存1比特位的基本机制，就可以轻松地构建任意位宽的寄存器，这可以通过由多个1比特位寄存器构成的阵列来实现，以构建可保存多比特位值的寄存器（。此类寄存器的基本设计参数是它的宽度，即它所保存比特位的数量—比如，16、32或64。此类寄存器的多位容量通常用字（word）来表示。

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/june/p18.png)

### 内存（Memories）
一旦具备了表示字的基本能力，就可以构建任意长度的存储块了。我们可以通过将很多寄存器堆叠起来形成随机存取 RAM 单元来实现。

随机存取内存（RAM,Random Access Memory）这个概念由此得来：在RAM上能够随机访问被选择的字而不会受限于访问顺序。也就是说，我们要求内存中的任何字（无论具体物理位
置在哪里）都能以相等的速度被直接访问。

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/june/p19.png)

这个规定可以通过以下方式实现。首先，根据其各自被存取的位置,也就是n位 RAM 中每个记忆单元—分配一个唯一的地址（address，一个从0到n一1之间的整数）。然后，构建一个由n-寄存器构成的阵列，再构建一个逻辑门，使得该逻辑门能够根据给定的地址 j，找到地址 j 对应的记忆单元。

要注意，由于记忆单元没有以物理实现的方式被标上地址，因此地址概念并没有在 RAM设计中显式地表现出来，而我们在后面将会看到，当芯片配备了直接访问逻辑部件后，就能以逻辑方式实现寻址的概念。

总的来说，典型的RAM设备接收三种输入：数据输入、地址输入和加载位。地址指定了在当前时种周期里哪一个 RAM 寄存器被访问。

进行读操作时（Load = 0），RAM的输出立即发出被选中的记忆单元的值。

在进行写操作（load = 1）时，被选择的记忆单元将在下一个时间周期内被赋予新输入值，从此 RAM 将开始发出该新输入值。

### 计数器（Counter）
计数器是一种时序芯片，它的状态是整数，每经过一个时间周期，该控数就增加1个单位，执行函数our（t）=our（t-1）+C，这里c就是1。

计数器在数字系统中担当非常重要的角色。比如，典型的CPU 包括一个程序计数器 （program counter），它的输出就是当前程序中下一步将要执行的指令地址。

计数器芯片可以通过将标准寄存器的输入/输出逻辑和附加的常数组合逻辑相结合来实现。一般来说，计数器必须要配一些附加的功能块，比如将计数置零、加载新的计数值，
或者用减操作来取代增操作。

## 项目
本章介绍的所有内存系统的基础是触发器(Flip-Flops)，并将这种门电路看作是原始的构建模块。在硬件知识的教材中，通常是从基本的组合逻辑门（比如 Nand 门）开始讲述，使用合适的反馈环来实现触发器。本书将DFF 看作是原始门，因此它们可以直接被用在硬件构造项目里面，而不用去管这些 DFF 的内部结构。

触发器 DFF 门能够利用第1章中介绍的一些底层级逻辑门来实现。然而，本书将DFF 看作是原始门，

### Bit
```
/**
 * 1-bit register:
 * If load is asserted, the register's value is set to in;
 * Otherwise, the register maintains its current value:
 * if (load(t)) out(t+1) = in(t), else out(t+1) = out(t)
 */
CHIP Bit {
    IN in, load;
    OUT out;

    PARTS:
    Mux(a = preOut, b = in, sel = load, out = o1);
    DFF(in = o1, out = preOut, out = out);
}
```

### Register
```
/**
 * 16-bit register:
 * If load is asserted, the register's value is set to in;
 * Otherwise, the register maintains its current value:
 * if (load(t)) out(t+1) = int(t), else out(t+1) = out(t)
 */
CHIP Register {
    IN in[16], load;
    OUT out[16];

    PARTS:
    Bit(in = in[0], load = load, out = out[0]);
    Bit(in = in[1], load = load, out = out[1]);
    Bit(in = in[2], load = load, out = out[2]);
    Bit(in = in[3], load = load, out = out[3]);
    Bit(in = in[4], load = load, out = out[4]);
    Bit(in = in[5], load = load, out = out[5]);
    Bit(in = in[6], load = load, out = out[6]);
    Bit(in = in[7], load = load, out = out[7]);
    Bit(in = in[8], load = load, out = out[8]);
    Bit(in = in[9], load = load, out = out[9]);
    Bit(in = in[10], load = load, out = out[10]);
    Bit(in = in[11], load = load, out = out[11]);
    Bit(in = in[12], load = load, out = out[12]);
    Bit(in = in[13], load = load, out = out[13]);
    Bit(in = in[14], load = load, out = out[14]);
    Bit(in = in[15], load = load, out = out[15]);
}
```

### RAM8
```
/**
 * Memory of eight 16-bit registers.
 * If load is asserted, the value of the register selected by
 * address is set to in; Otherwise, the value does not change.
 * The value of the selected register is emitted by out.
 */
CHIP RAM8 {
    IN in[16], load, address[3];
    OUT out[16];

    PARTS:
    DMux8Way(in = load, sel = address, a = loadA, b = loadB, c = loadC, d = loadD, e = loadE, f = loadF, g = loadG, h = loadH);

    Register(in = in, load = loadA, out = o1);
    Register(in = in, load = loadB, out = o2);
    Register(in = in, load = loadC, out = o3);
    Register(in = in, load = loadD, out = o4);
    Register(in = in, load = loadE, out = o5);
    Register(in = in, load = loadF, out = o6);
    Register(in = in, load = loadG, out = o7);
    Register(in = in, load = loadH, out = o8);

    Mux8Way16(a = o1, b = o2, c = o3, d = o4, e = o5, f = o6, g = o7, h = o8, sel = address, out = out);
}
```

### RAM64
```
/**
 * Memory of sixty four 16-bit registers.
 * If load is asserted, the value of the register selected by
 * address is set to in; Otherwise, the value does not change.
 * The value of the selected register is emitted by out.
 */
CHIP RAM64 {
    IN in[16], load, address[6];
    OUT out[16];

    PARTS:
    DMux8Way(in = load, sel = address[3..5], a = loadA, b = loadB, c = loadC, d = loadD, e = loadE, f = loadF, g = loadG, h = loadH);

    RAM8(in = in, load = loadA, address = address[0..2], out = o1);
    RAM8(in = in, load = loadB, address = address[0..2], out = o2);
    RAM8(in = in, load = loadC, address = address[0..2], out = o3);
    RAM8(in = in, load = loadD, address = address[0..2], out = o4);
    RAM8(in = in, load = loadE, address = address[0..2], out = o5);
    RAM8(in = in, load = loadF, address = address[0..2], out = o6);
    RAM8(in = in, load = loadG, address = address[0..2], out = o7);
    RAM8(in = in, load = loadH, address = address[0..2], out = o8);

    Mux8Way16(a = o1, b = o2, c = o3, d = o4, e = o5, f = o6, g = o7, h = o8, sel = address[3..5], out = out);
}
```

### RAM512
```
/**
 * Memory of 512 16-bit registers.
 * If load is asserted, the value of the register selected by
 * address is set to in; Otherwise, the value does not change.
 * The value of the selected register is emitted by out.
 */
CHIP RAM512 {
    IN in[16], load, address[9];
    OUT out[16];

    PARTS:
    DMux8Way(in = load, sel = address[6..8], a = loadA, b = loadB, c = loadC, d = loadD, e = loadE, f = loadF, g = loadG, h = loadH);

    RAM64(in = in, load = loadA, address = address[0..5], out = o1);
    RAM64(in = in, load = loadB, address = address[0..5], out = o2);
    RAM64(in = in, load = loadC, address = address[0..5], out = o3);
    RAM64(in = in, load = loadD, address = address[0..5], out = o4);
    RAM64(in = in, load = loadE, address = address[0..5], out = o5);
    RAM64(in = in, load = loadF, address = address[0..5], out = o6);
    RAM64(in = in, load = loadG, address = address[0..5], out = o7);
    RAM64(in = in, load = loadH, address = address[0..5], out = o8);

    Mux8Way16(a = o1, b = o2, c = o3, d = o4, e = o5, f = o6, g = o7, h = o8, sel = address[6..8], out = out);
}
```

### RAM4K
```
/**
 * Memory of 4K 16-bit registers.
 * If load is asserted, the value of the register selected by
 * address is set to in; Otherwise, the value does not change.
 * The value of the selected register is emitted by out.
 */
CHIP RAM4K {
    IN in[16], load, address[12];
    OUT out[16];

    PARTS:
    DMux8Way(in = load, sel = address[9..11], a = loadA, b = loadB, c = loadC, d = loadD, e = loadE, f = loadF, g = loadG, h = loadH);

    RAM512(in = in, load = loadA, address = address[0..8], out = o1);
    RAM512(in = in, load = loadB, address = address[0..8], out = o2);
    RAM512(in = in, load = loadC, address = address[0..8], out = o3);
    RAM512(in = in, load = loadD, address = address[0..8], out = o4);
    RAM512(in = in, load = loadE, address = address[0..8], out = o5);
    RAM512(in = in, load = loadF, address = address[0..8], out = o6);
    RAM512(in = in, load = loadG, address = address[0..8], out = o7);
    RAM512(in = in, load = loadH, address = address[0..8], out = o8);

    Mux8Way16(a = o1, b = o2, c = o3, d = o4, e = o5, f = o6, g = o7, h = o8, sel = address[9..11], out = out);
}
```

### RAM16K
```
CHIP RAM16K {
    IN in[16], load, address[14];
    OUT out[16];

    PARTS:
    DMux4Way(in = load, sel = address[12..13], a = loadA, b = loadB, c = loadC, d = loadD);

    RAM4K(in = in, load = loadA, address = address[0..11], out = o1);
    RAM4K(in = in, load = loadB, address = address[0..11], out = o2);
    RAM4K(in = in, load = loadC, address = address[0..11], out = o3);
    RAM4K(in = in, load = loadD, address = address[0..11], out = o4);

    Mux4Way16(a = o1, b = o2, c = o3, d = o4, sel = address[12..13], out = out);
}
```

### PC
```
/**
 * A 16-bit counter.
 * if      reset(t): out(t+1) = 0
 * else if load(t):  out(t+1) = in(t)
 * else if inc(t):   out(t+1) = out(t) + 1
 * else              out(t+1) = out(t)
 */
CHIP PC {
    IN in[16], reset, load, inc;
    OUT out[16];
    
    PARTS:
    Inc16(in = preOut, out = addOut);
    
    Mux16(a = preOut, b = addOut, sel = inc, out = o1);

    Mux16(a = o1, b = in, sel = load, out = o2);

    Mux16(a = o2, b = false, sel = reset, out = o3);

    Register(in = o3, load = true, out = preOut, out = out);
}
```