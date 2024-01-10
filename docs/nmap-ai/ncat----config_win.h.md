# `nmap\ncat\config_win.h`

```
/* $Id: ncat.h 16595 2010-01-27 02:51:16Z fyodor $ */
/* 这些是在 Windows 上生效的预处理器定义，因为没有 Autoconf 来创建 config.h。 */

// 定义是否有 OpenSSL
#define HAVE_OPENSSL 1
// 定义是否有 SSL_SET_TLSEXT_HOST_NAME
#define HAVE_SSL_SET_TLSEXT_HOST_NAME 1
// 定义是否有 HTTP_DIGEST
#define HAVE_HTTP_DIGEST 1
// 定义是否包含 Lua
#define LUA_INCLUDED 1
// 定义是否有 Lua
#define HAVE_LUA 1
```