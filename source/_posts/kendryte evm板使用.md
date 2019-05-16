---
title: kendryte evm板使用
date: 2019-04-18 23:11:40
tags: [MCU, k210]
---

## 1.问题解决

### 1.1 工具链和 cmake 的 cache 问题

工具链和当前的开发分支，硬件和软件浮点不匹配，可以通过修改工程编译选项解决

<pre class="themepre">
can't link hard-float modules with soft-float module
</pre>

修改工程编译选项之后，因为 cmake 的 cache 问题，可能重新 cmake 也不能生成新的编译选项，可以选择删掉 CMakeCache.txt 文件

### 1.2 板子上的串口 CH340 只能用来升级，发现无法正常输出

直接把 TX 和 RX （IO04 和 IO05）接出来接自己的 USB 转串口工具，ISP 升级的时候就是 IO16 需要在 RESET 的时候接地。然后使用 k-flash 的时候使用波特率 115200，不然可能超时错误。