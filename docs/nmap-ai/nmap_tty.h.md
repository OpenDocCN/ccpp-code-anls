# `nmap\nmap_tty.h`

```cpp
// 如果 NMAP_TTY_H 未定义，则定义 NMAP_TTY_H
#ifndef NMAP_TTY_H
#define NMAP_TTY_H

/*
 * 初始化终端以进行无缓冲非阻塞输入。还通过 atexit() 注册 tty_done()。在调用 keyWasPressed() 之前，需要调用此函数。
 */
void tty_init();

/* 捕获所有预定义的按键并解释它们，还会告诉你是否应该打印任何内容。返回 true 表示已按下非标准按键，调用方法应打印状态消息 */
bool keyWasPressed();

#endif
```