# `nmap\libpcre\src\pcre2_serialize.c`

```cpp
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2016-2020 University of Cambridge

----------------------------------------------------------------------------- 
Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

    * Redistributions of source code must retain the above copyright notice,
      this list of conditions and the following disclaimer.

    * Redistributions in binary form must reproduce the above copyright
      notice, this list of conditions and the following disclaimer in the
      documentation and/or other materials provided with the distribution.

    * Neither the name of the University of Cambridge nor the names of its
      contributors may be used to endorse or promote products derived from
      this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
/*
This module contains functions for serializing and deserializing a sequence of compiled codes.
*/

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcre2_internal.h"

/* Magic number to provide a small check against being handed junk. */
#define SERIALIZED_DATA_MAGIC 0x50523253u

/* Deserialization is limited to the current PCRE version and character width. */
#define SERIALIZED_DATA_VERSION ((PCRE2_MAJOR) | ((PCRE2_MINOR) << 16))
#define SERIALIZED_DATA_CONFIG (sizeof(PCRE2_UCHAR) | ((sizeof(void*)) << 8) | ((sizeof(PCRE2_SIZE)) << 16))

/*************************************************
*           Serialize compiled patterns          *
*************************************************/
PCRE2_EXP_DEFN int32_t PCRE2_CALL_CONVENTION
pcre2_serialize_encode(const pcre2_code **codes, int32_t number_of_codes,
   uint8_t **serialized_bytes, PCRE2_SIZE *serialized_size,
   pcre2_general_context *gcontext)
{
  uint8_t *bytes;
  uint8_t *dst_bytes;
  int32_t i;
  PCRE2_SIZE total_size;
  const pcre2_real_code *re;
  const uint8_t *tables;
  pcre2_serialized_data *data;

  const pcre2_memctl *memctl = (gcontext != NULL) ? &gcontext->memctl : &PRIV(default_compile_context).memctl;

  if (codes == NULL || serialized_bytes == NULL || serialized_size == NULL)
    return PCRE2_ERROR_NULL;

  if (number_of_codes <= 0) return PCRE2_ERROR_BADDATA;

  /* Compute total size. */
  total_size = sizeof(pcre2_serialized_data) + TABLES_LENGTH;
  tables = NULL;

  for (i = 0; i < number_of_codes; i++)
  {
    if (codes[i] == NULL) return PCRE2_ERROR_NULL;
    re = (const pcre2_real_code *)(codes[i]);
    if (re->magic_number != MAGIC_NUMBER) return PCRE2_ERROR_BADMAGIC;
    if (tables == NULL)
      tables = re->tables;
    else if (tables != re->tables)
    # 返回 PCRE2_ERROR_MIXEDTABLES 错误代码
    return PCRE2_ERROR_MIXEDTABLES;
  # 将 re->blocksize 的值加到 total_size 上
  total_size += re->blocksize;
  # 结束循环
  }
/* 初始化字节流。 */
bytes = memctl->malloc(total_size + sizeof(pcre2_memctl), memctl->memory_data);
if (bytes == NULL) return PCRE2_ERROR_NOMEMORY;

/* 控制器被存储为一个隐藏参数。 */
memcpy(bytes, memctl, sizeof(pcre2_memctl));
bytes += sizeof(pcre2_memctl);

data = (pcre2_serialized_data *)bytes;
data->magic = SERIALIZED_DATA_MAGIC;
data->version = SERIALIZED_DATA_VERSION;
data->config = SERIALIZED_DATA_CONFIG;
data->number_of_codes = number_of_codes;

/* 复制所有编译代码数据。 */
dst_bytes = bytes + sizeof(pcre2_serialized_data);
memcpy(dst_bytes, tables, TABLES_LENGTH);
dst_bytes += TABLES_LENGTH;

for (i = 0; i < number_of_codes; i++)
  {
  re = (const pcre2_real_code *)(codes[i]);
  (void)memcpy(dst_bytes, (char *)re, re->blocksize);
  
  /* 在反序列化期间，编译代码块中的某些字段将被重新设置。
  为了确保相同模式的序列化数据流始终相同，这里将它们设置为零。
  我们不能假设模式的副本对于访问字段作为结构的一部分是正确对齐的。
  注意在第二个memset中使用sizeof(void *)，以指定指针的大小。
  如果使用sizeof(uint8_t *)（tables是指向uint8_t的指针），gcc会发出警告，因为第一个参数也是指向uint8_t的指针。
  将第一个参数强制转换为(void *)可以阻止这种情况，但这并没有阻止Coverity发出相同的投诉。 */

  (void)memset(dst_bytes + offsetof(pcre2_real_code, memctl), 0, 
    sizeof(pcre2_memctl));
  (void)memset(dst_bytes + offsetof(pcre2_real_code, tables), 0, 
    sizeof(void *));
  (void)memset(dst_bytes + offsetof(pcre2_real_code, executable_jit), 0,
    sizeof(void *));        
 
  dst_bytes += re->blocksize;
  }

*serialized_bytes = bytes;
*serialized_size = total_size;
return number_of_codes;
}
/*************************************************/

PCRE2_EXP_DEFN int32_t PCRE2_CALL_CONVENTION
pcre2_serialize_decode(pcre2_code **codes, int32_t number_of_codes,
   const uint8_t *bytes, pcre2_general_context *gcontext)
{
const pcre2_serialized_data *data = (const pcre2_serialized_data *)bytes;  // 将字节流转换为序列化数据结构
const pcre2_memctl *memctl = (gcontext != NULL) ?  // 根据传入的上下文确定内存控制器
  &gcontext->memctl : &PRIV(default_compile_context).memctl;

const uint8_t *src_bytes;  // 定义源字节流指针
pcre2_real_code *dst_re;  // 定义目标编译后的正则表达式指针
uint8_t *tables;  // 定义表格指针
int32_t i, j;  // 定义循环变量

/* Sanity checks. */

if (data == NULL || codes == NULL) return PCRE2_ERROR_NULL;  // 检查传入的数据和代码是否为空
if (number_of_codes <= 0) return PCRE2_ERROR_BADDATA;  // 检查代码数量是否合法
if (data->number_of_codes <= 0) return PCRE2_ERROR_BADSERIALIZEDDATA;  // 检查序列化数据中的代码数量是否合法
if (data->magic != SERIALIZED_DATA_MAGIC) return PCRE2_ERROR_BADMAGIC;  // 检查序列化数据的魔数是否正确
if (data->version != SERIALIZED_DATA_VERSION) return PCRE2_ERROR_BADMODE;  // 检查序列化数据的版本是否正确
if (data->config != SERIALIZED_DATA_CONFIG) return PCRE2_ERROR_BADMODE;  // 检查序列化数据的配置是否正确

if (number_of_codes > data->number_of_codes)  // 如果传入的代码数量大于序列化数据中的代码数量
  number_of_codes = data->number_of_codes;  // 则将代码数量设为序列化数据中的代码数量

src_bytes = bytes + sizeof(pcre2_serialized_data);  // 源字节流指针指向序列化数据之后的位置

/* Decode tables. The reference count for the tables is stored immediately
following them. */

tables = memctl->malloc(TABLES_LENGTH + sizeof(PCRE2_SIZE), memctl->memory_data);  // 分配内存用于存储表格
if (tables == NULL) return PCRE2_ERROR_NOMEMORY;  // 如果内存分配失败，则返回内存不足错误

memcpy(tables, src_bytes, TABLES_LENGTH);  // 将源字节流中的表格数据复制到分配的内存中
*(PCRE2_SIZE *)(tables + TABLES_LENGTH) = number_of_codes;  // 将代码数量存储在表格数据之后的位置
src_bytes += TABLES_LENGTH;  // 源字节流指针移动到表格数据之后的位置

/* Decode the byte stream. We must not try to read the size from the compiled
code block in the stream, because it might be unaligned, which causes errors on
hardware such as Sparc-64 that doesn't like unaligned memory accesses. The type
of the blocksize field is given its own name to ensure that it is the same here
as in the block. */

for (i = 0; i < number_of_codes; i++)  // 遍历每个代码块
  {
  CODE_BLOCKSIZE_TYPE blocksize;  // 定义代码块大小变量
  memcpy(&blocksize, src_bytes + offsetof(pcre2_real_code, blocksize),  // 从源字节流中复制代码块大小
    sizeof(CODE_BLOCKSIZE_TYPE));
  if (blocksize <= sizeof(pcre2_real_code))  // 如果代码块大小小于等于编译后的正则表达式的大小
    # 返回错误代码，指示序列化数据损坏
    return PCRE2_ERROR_BADSERIALIZEDDATA;

  # 由 gcontext 提供的分配器替换了原始的分配器
  dst_re = (pcre2_real_code *)PRIV(memctl_malloc)(blocksize,
    (pcre2_memctl *)gcontext);
  if (dst_re == NULL)
    {
    memctl->free(tables, memctl->memory_data);
    for (j = 0; j < i; j++)
      {
      memctl->free(codes[j], memctl->memory_data);
      codes[j] = NULL;
      }
    # 返回内存不足错误代码
    return PCRE2_ERROR_NOMEMORY;
    }

  # 必须保留新的分配器
  memcpy(((uint8_t *)dst_re) + sizeof(pcre2_memctl),
    src_bytes + sizeof(pcre2_memctl), blocksize - sizeof(pcre2_memctl));
  if (dst_re->magic_number != MAGIC_NUMBER ||
      dst_re->name_entry_size > MAX_NAME_SIZE + IMM2_SIZE + 1 ||
      dst_re->name_count > MAX_NAME_COUNT)
    {   
    memctl->free(dst_re, memctl->memory_data); 
    # 返回错误代码，指示序列化数据损坏
    return PCRE2_ERROR_BADSERIALIZEDDATA;
    } 

  # 目前只支持一个表
  dst_re->tables = tables;
  dst_re->executable_jit = NULL;
  dst_re->flags |= PCRE2_DEREF_TABLES;

  codes[i] = dst_re;
  src_bytes += blocksize;
  }
# 返回已序列化模式的数量
return number_of_codes;
}


/*************************************************
*    获取序列化模式的数量                         *
*************************************************/

PCRE2_EXP_DEFN int32_t PCRE2_CALL_CONVENTION
pcre2_serialize_get_number_of_codes(const uint8_t *bytes)
{
# 将字节流转换为序列化数据结构
const pcre2_serialized_data *data = (const pcre2_serialized_data *)bytes;

# 如果数据为空，则返回空指针错误
if (data == NULL) return PCRE2_ERROR_NULL;
# 如果数据的魔术数不匹配，则返回错误
if (data->magic != SERIALIZED_DATA_MAGIC) return PCRE2_ERROR_BADMAGIC;
# 如果数据的版本不匹配，则返回错误
if (data->version != SERIALIZED_DATA_VERSION) return PCRE2_ERROR_BADMODE;
# 如果数据的配置不匹配，则返回错误
if (data->config != SERIALIZED_DATA_CONFIG) return PCRE2_ERROR_BADMODE;

# 返回数据中的模式数量
return data->number_of_codes;
}


/*************************************************
*            释放已分配的流                       *
*************************************************/

PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_serialize_free(uint8_t *bytes)
{
# 如果字节流不为空
if (bytes != NULL)
  {
  # 从字节流中获取内存控制块
  pcre2_memctl *memctl = (pcre2_memctl *)(bytes - sizeof(pcre2_memctl));
  # 释放内存控制块中的内存
  memctl->free(memctl, memctl->memory_data);
  }
}

/* pcre2_serialize.c 的结束 */
```