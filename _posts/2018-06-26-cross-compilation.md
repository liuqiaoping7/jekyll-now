---
layout: post
title: Embedded system & Cross Compilation
---
以下我们选取TI公司的DM814x/AM387x EVM Baseboard为平台来讨论嵌入式系统、交叉编译环境的一些常识。

1、背景
1.1、嵌入式系统
这是维基百科词条，说明了嵌入式系统的特性、常见的试用领域：
An embedded system is a computer system with a dedicated function within a larger mechanical or electrical system, often with real-time computing constraints. It is embedded as part of a complete device often including hardware and mechanical parts. Embedded systems control many devices in common use today. Ninety-eight percent of all microprocessors are manufactured as components of embedded systems.
Examples of properties of typical embedded computers when compared with general-purpose counterparts are low power consumption, small size, rugged operating ranges, and low per-unit cost. This comes at the price of limited processing resources, which make them significantly more difficult to program and to interact with. However, by building intelligence mechanisms on top of the hardware, taking advantage of possible existing sensors and the existence of a network of embedded units, one can both optimally manage available resources at the unit and network levels as well as provide augmented functions, well beyond those available.For example, intelligent techniques can be designed to manage power consumption of embedded systems.
Modern embedded systems are often based on microcontrollers (i.e. CPUs with integrated memory or peripheral interfaces), but ordinary microprocessors (using external chips for memory and peripheral interface circuits) are also common, especially in more-complex systems. In either case, the processor(s) used may be types ranging from general purpose to those specialized in certain class of computations, or even custom designed for the application at hand. A common standard class of dedicated processors is the digital signal processor (DSP).
Since the embedded system is dedicated to specific tasks, design engineers can optimize it to reduce the size and cost of the product and increase the reliability and performance. Some embedded systems are mass-produced, benefiting from economies of scale.
Embedded systems range from portable devices such as digital watches and MP3 players, to large stationary installations like traffic lights, factory controllers, and largely complex systems like hybrid vehicles, MRI, and avionics. Complexity varies from low, with a single microcontroller chip, to very high with multiple units, peripherals and networks mounted inside a large chassis or enclosure.

1.2、交叉编译
这是维基百科词条，说明了交叉编译的定义、存在的意义：
A cross compiler is a compiler capable of creating executable code for a platform other than the one on which the compiler is running. For example, a compiler that runs on a Windows 7 PC but generates code that runs on Android smartphone is a cross compiler.
A cross compiler is necessary to compile code for multiple platforms from one development host. Direct compilation on the target platform might be infeasible, for example on a microcontroller of an embedded system, because those systems contain no operating system. In paravirtualization, one computer runs multiple operating systems and a cross compiler could generate an executable for each of them from one main source.

2、获取资料
通常芯片公司推出一款芯片都会给出说明特性、开发套件。见 http://www.ti.com/product/TMS320DM8148/ 。
说明特性
通常是给二次开发的项目负责人看的。项目负责人的职能是面对具体的问题或者需求找到合适的解决方案，而说明特性就是告诉他们如果你碰到了以下领域的问题或者以下指标需求，请选择我。见 http://www.ti.com/product/TMS320DM8148/description#descriptions 。
开发套件
通常是给二次开发的项目工程师看的。通常以评估板卡来展示外围接口扩展、以示例软件来展示实际业务能力。见 http://www.ti.com/product/TMS320DM8148/toolssoftware 链接至 http://www.ti.com/tool/linuxezsdk-davinci 。
以下我们将以项目工程师的角色进行实际业务二次开发。

3、交叉编译环境构建

4、二次开发
