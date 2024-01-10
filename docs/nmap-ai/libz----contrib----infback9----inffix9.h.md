# `nmap\libz\contrib\infback9\inffix9.h`

```
    /* inffix9.h -- table for decoding deflate64 fixed codes
     * Generated automatically by makefixed9().
     */
    /* 定义了用于解码deflate64固定代码的表
     * 由makefixed9()自动生成。
     */
    
    /* WARNING: this file should *not* be used by applications.
       It is part of the implementation of this library and is
       subject to change. Applications should only use zlib.h.
     */
    /* 警告：此文件不应被应用程序使用。
       它是这个库的实现的一部分，可能会发生变化。应用程序应该只使用zlib.h。
     */
    
    };
    
    static const code distfix[32] = {
        {128,5,1},{135,5,257},{131,5,17},{139,5,4097},{129,5,5},
        {137,5,1025},{133,5,65},{141,5,16385},{128,5,3},{136,5,513},
        {132,5,33},{140,5,8193},{130,5,9},{138,5,2049},{134,5,129},
        {142,5,32769},{128,5,2},{135,5,385},{131,5,25},{139,5,6145},
        {129,5,7},{137,5,1537},{133,5,97},{141,5,24577},{128,5,4},
        {136,5,769},{132,5,49},{140,5,12289},{130,5,13},{138,5,3073},
        {134,5,193},{142,5,49153}
    };
    /* 定义了一个包含32个元素的固定数组distfix
     * 每个元素是一个code结构，包含三个值
     * 这个数组用于解码deflate64固定代码
     */
```