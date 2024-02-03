# `nmap\libz\inffixed.h`

```cpp
    /* inffixed.h -- table for decoding fixed codes
     * Generated automatically by makefixed().
     */
    // 用于解码固定代码的表格，由 makefixed() 自动生成

    /* WARNING: this file should *not* be used by applications.
       It is part of the implementation of this library and is
       subject to change. Applications should only use zlib.h.
     */
    // 警告：此文件不应被应用程序使用。它是这个库的实现的一部分，可能会发生变化。应用程序应该只使用 zlib.h。

    };

    static const code distfix[32] = {
        {16,5,1},{23,5,257},{19,5,17},{27,5,4097},{17,5,5},{25,5,1025},
        {21,5,65},{29,5,16385},{16,5,3},{24,5,513},{20,5,33},{28,5,8193},
        {18,5,9},{26,5,2049},{22,5,129},{64,5,0},{16,5,2},{23,5,385},
        {19,5,25},{27,5,6145},{17,5,7},{25,5,1537},{21,5,97},{29,5,24577},
        {16,5,4},{24,5,769},{20,5,49},{28,5,12289},{18,5,13},{26,5,3073},
        {22,5,193},{64,5,0}
    };
    // 固定代码的解码表
```