# `nmap\libssh2\nw\keepscreen.c`

```
/* Simple _NonAppStop() implementation which can be linked to your 
 * NLM in order to keep the screen open when the NLM terminates
 * (the good old clib behaviour).
 * You dont have to call it, its done automatically from LibC.
 *
 * 2004-Aug-11  by Guenter Knauf 
 *
 * URL: http://www.gknw.net/development/mk_nlm/
 */
 
#include <stdio.h>
#include <screen.h>

void _NonAppStop()
{
    uint16_t row, col;
    
    // 获取屏幕的行数和列数
    GetScreenSize(&row, &col);
    // 移动光标到最后一行第一列
    gotorowcol(row-1, 0);
    /* pressanykey(); */
    // 打印提示信息
    printf("<Press any key to close screen> ");
    // 获取用户输入的字符
    getcharacter();
}
```