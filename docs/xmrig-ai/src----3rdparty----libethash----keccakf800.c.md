# `xmrig\src\3rdparty\libethash\keccakf800.c`

```cpp
/* ethash: C/C++ implementation of Ethash, the Ethereum Proof of Work algorithm.
 * Copyright 2018-2019 Pawel Bylica.
 * Licensed under the Apache License, Version 2.0.
 */

#include <stdint.h>

// 定义一个左循环移位函数，将 x 左移 s 位，或者右移 32-s 位
static uint32_t rol(uint32_t x, unsigned s)
{
    return (x << s) | (x >> (32 - s));
}

// 定义一个包含22个常量的数组
static const uint32_t round_constants[22] = {
    0x00000001,
    0x00008082,
    0x0000808A,
    0x80008000,
    0x0000808B,
    0x80000001,
    0x80008081,
    0x00008009,
    0x0000008A,
    0x00000088,
    0x80008009,
    0x8000000A,
    0x8000808B,
    0x0000008B,
    0x00008089,
    0x00008003,
    0x00008002,
    0x00000080,
    0x0000800A,
    0x8000000A,
    0x80008081,
    0x00008080,
};

// 定义一个函数，对输入的 25 个 32 位无符号整数进行 Keccak-800 算法处理
void ethash_keccakf800(uint32_t state[25])
{
    /* The implementation directly translated from ethash_keccakf1600. */

    int round;

    uint32_t Aba, Abe, Abi, Abo, Abu;
    uint32_t Aga, Age, Agi, Ago, Agu;
    uint32_t Aka, Ake, Aki, Ako, Aku;
    uint32_t Ama, Ame, Ami, Amo, Amu;
    uint32_t Asa, Ase, Asi, Aso, Asu;

    uint32_t Eba, Ebe, Ebi, Ebo, Ebu;
    uint32_t Ega, Ege, Egi, Ego, Egu;
    uint32_t Eka, Eke, Eki, Eko, Eku;
    uint32_t Ema, Eme, Emi, Emo, Emu;
    uint32_t Esa, Ese, Esi, Eso, Esu;

    uint32_t Ba, Be, Bi, Bo, Bu;

    uint32_t Da, De, Di, Do, Du;

    // 将输入状态的值赋给对应的变量
    Aba = state[0];
    Abe = state[1];
    Abi = state[2];
    Abo = state[3];
    Abu = state[4];
    Aga = state[5];
    Age = state[6];
    Agi = state[7];
    Ago = state[8];
    Agu = state[9];
    Aka = state[10];
    Ake = state[11];
    Aki = state[12];
    Ako = state[13];
    Aku = state[14];
    Ama = state[15];
    Ame = state[16];
    Ami = state[17];
    Amo = state[18];
    Amu = state[19];
    Asa = state[20];
    Ase = state[21];
    Asi = state[22];
    Aso = state[23];
    Asu = state[24];

    // 循环进行22轮的计算
    for (round = 0; round < 22; round += 2)
    }

    // 将计算后的状态值赋回输入状态数组
    state[0] = Aba;
    state[1] = Abe;
    state[2] = Abi;
    state[3] = Abo;
    state[4] = Abu;
    state[5] = Aga;
    state[6] = Age;
    state[7] = Agi;
    # 将特定值赋给数组 state 的不同索引位置
    state[8] = Ago;
    state[9] = Agu;
    state[10] = Aka;
    state[11] = Ake;
    state[12] = Aki;
    state[13] = Ako;
    state[14] = Aku;
    state[15] = Ama;
    state[16] = Ame;
    state[17] = Ami;
    state[18] = Amo;
    state[19] = Amu;
    state[20] = Asa;
    state[21] = Ase;
    state[22] = Asi;
    state[23] = Aso;
    state[24] = Asu;
# 闭合前面的函数定义
```