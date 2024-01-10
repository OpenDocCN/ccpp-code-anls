# `nmap\nping\NEPContext.h`

```
#ifndef __NEPCONTEXT_H__  // 如果未定义__NEPCONTEXT_H__，则执行以下代码
#define __NEPCONTEXT_H__ 1  // 定义__NEPCONTEXT_H__为1

#include "nsock.h"  // 包含nsock.h头文件
#include "EchoHeader.h"  // 包含EchoHeader.h头文件
#include <vector>  // 包含vector头文件

/* SERVER STATE MACHINE                                                       */
/*                      _                                                     */
/*                     (O)                                                    */
/*                      |                                                     */
/* Capture Raw /        |                  Rcvd TCP Connection /              */
/* Send NEP_ECHO        |     +--------+   Send NEP_HANDSHAKE_SERVER          */
/*   +----------+       +---->| LISTEN |-----------------+                    */
/*   |          |             +--------+                 |                    */
/*   |         \|/                                      \|/                   */
/*   |     +----+------+                      +----------+-----------+        */
/*   +-----| NEP_READY |                      | NEP_HANDSHAKE_SERVER |        */
/*         |    SENT   |                      |         SENT         |        */
/*         +----+------+                      +----------+-----------+        */
/*             /|\                                       |                    */
/*              |                                        |                    */
/*              |       +---------------------+          |                    */
/*              |       | NEP_HANDSHAKE_FINAL |          |                    */
/*              +-------|        SENT         |<---------+                    */
/*                      +---------------------+   Rcvd NEP_HANDSHAKE_CLIENT/  */
/*  Rcvd NEP_PACKETSPEC /                         Send NEP_HANDSHAKE_FINAL    */
/*  Send NEP_READY                                                            */
/*                                                                            */
/*                                                                            */
// 定义不同状态的常量
#define STATE_LISTEN          0x00
#define STATE_HS_SERVER_SENT  0x01
#define STATE_HS_FINAL_SENT   0x02
#define STATE_READY_SENT      0x03

// 客户端未找到的标识符
#define CLIENT_NOT_FOUND -1
typedef int clientid_t; /**< Type for client identifiers */

// 不同密钥类型的常量
#define MAC_KEY_S2C_INITIAL 0x01
#define MAC_KEY_S2C         0x02
#define MAC_KEY_C2S         0x03
#define CIPHER_KEY_C2S      0x04
#define CIPHER_KEY_S2C      0x05

/* 客户端字段说明符 */
typedef struct field_spec{
  u8 field; /* 字段标识符（参见 NEP RFC） */
  u8 len;   /* 字段长度 */
  u8 value[PACKETSPEC_FIELD_LEN]; /* 字段数据 */
}fspec_t;

class NEPContext {

    private:

        clientid_t id;     /**<  客户端标识符 */
        nsock_iod nsi;     /**<  客户端 nsock IOD  */
        int state;         // 客户端状态
        u32 last_seq_client; // 客户端最后一个序列号
        u32 last_seq_server; // 服务器最后一个序列号
        u8 next_iv_enc[CIPHER_BLOCK_SIZE]; // 下一个加密初始化向量
        u8 next_iv_dec[CIPHER_BLOCK_SIZE]; // 下一个解密初始化向量
        u8 nep_key_mac_c2s[MAC_KEY_LEN]; // 客户端到服务器的 NEP 密钥
        u8 nep_key_mac_s2c[MAC_KEY_LEN]; // 服务器到客户端的 NEP 密钥
        u8 nep_key_ciphertext_c2s[CIPHER_KEY_LEN]; // 客户端到服务器的密文密钥
        u8 nep_key_ciphertext_s2c[CIPHER_KEY_LEN]; // 服务器到客户端的密文密钥
        u8 server_nonce[NONCE_LEN]; // 服务器随机数
        u8 client_nonce[NONCE_LEN]; // 客户端随机数
        bool server_nonce_set; // 服务器随机数是否设置
        bool client_nonce_set; // 客户端随机数是否设置
        std::vector<fspec_t> fspecs; // 客户端字段说明符的向量
        struct sockaddr_storage clnt_addr; // 客户端地址

        u8 *generateKey(int key_type, size_t *final_len); // 生成密钥的函数

};

#endif /* __NEPCONTEXT_H__ */
```