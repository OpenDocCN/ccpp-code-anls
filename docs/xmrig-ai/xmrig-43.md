# xmrig源码解析 43

# `src/3rdparty/llhttp/api.c`

这段代码是一个C语言的函数，它包含了三个头文件：<stdlib.h>，<stdio.h>，<string.h>，以及一个自定义的header文件llhttp.h。

首先，<stdlib.h>和<stdio.h>头文件提供了标准库函数和输入/输出操作所需的支持，而<string.h>头文件则提供了字符串操作的支持。

接着，自定义的header文件llhttp.h中可能包含了与这段代码相关的函数和数据结构，但具体作用取决于这个header文件的内容。

而代码的主要作用是定义了一个名为"llhttp_settings_t"的类型，它指针了一个llhttp_settings_t结构体，并且在结构体中包含了一个名为"NAME"的成员和一个名为"SETTINGS"的成员。

进一步分析，可以看到这个llhttp_settings_t结构体可能被用来实现一个输入/输出parser的回调函数，通过提供NAME属性的值来告知parser设置请求的参数。而设置请求的参数可能是一个字符串，这个字符串会被用llhttp_settings_t结构体中NAME属性的值进行替换。


```cpp
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

#include "llhttp.h"

#define CALLBACK_MAYBE(PARSER, NAME, ...)                                     \
  do {                                                                        \
    const llhttp_settings_t* settings;                                        \
    settings = (const llhttp_settings_t*) (PARSER)->settings;                 \
    if (settings == NULL || settings->NAME == NULL) {                         \
      err = 0;                                                                \
      break;                                                                  \
    }                                                                         \
    err = settings->NAME(__VA_ARGS__);                                        \
  } while (0)

```



这段代码定义了一个名为`llhttp_init`的函数，它接受一个`llhttp_t`类型的参数`parser`，一个指向`llhttp_settings_t`类型的参数`settings`，以及一个指向`void`类型参数的指针`type`。

函数内部先调用`llhttp__internal_init`函数，确保在函数内部初始化`parser`时使用正确的设置。然后将`parser->type`和`parser->settings`都指向传入的`type`和`settings`。

如果定义了`__wasm__`宏，则需要在函数前添加`extern`关键字，以便在wasm中运行时确保函数正确声明。

接下来是可选的函数，它们用于处理http请求的响应。这些函数分别处理响应中的`type`字段、`url`字段、`status`字段和`header_field`字段。由于这些函数没有定义，它们的实现将取决于使用http协议的具体类型和响应格式。


```cpp
void llhttp_init(llhttp_t* parser, llhttp_type_t type,
                 const llhttp_settings_t* settings) {
  llhttp__internal_init(parser);

  parser->type = type;
  parser->settings = (void*) settings;
}


#if defined(__wasm__)

extern int wasm_on_message_begin(llhttp_t * p);
extern int wasm_on_url(llhttp_t* p, const char* at, size_t length);
extern int wasm_on_status(llhttp_t* p, const char* at, size_t length);
extern int wasm_on_header_field(llhttp_t* p, const char* at, size_t length);
```

这是一个llhttp库的定义，它定义了四个函数以及一个名为wasm_settings的常量。

这些函数是用于处理HTTP请求的，它们在llhttp库中起着重要的作用。

具体来说，这些函数的实现如下：

- wasm_on_header_value函数接收一个指向llhttp包的指针，一个HTTP头指针和一个字符数组。它使用llhttp库中的wasm_on_header_field函数来查找并提取HTTP头中的指定字段，然后使用wasm_on_header_value函数来获取该字段的值。最后，它将提取出的值返回到调用者。

- wasm_on_headers_complete函数接收一个指向llhttp包的指针和一个字符数组。它使用llhttp库中的wasm_on_headers_complete函数来处理接收到的HTTP头。然后，它将使用wasm_on_body函数和wasm_on_message_complete函数来处理请求的主体和消息。最后，它将使用wasm_on_message_complete函数来处理消息的结束。

- wasm_on_body函数接收一个指向llhttp包的指针和一个字符数组。它使用llhttp库中的wasm_on_body函数来处理请求的主体。然后，它将使用wasm_on_message_complete函数来处理消息的结束。

- wasm_on_message_complete函数接收一个指向llhttp包的指针和一个字符数组。它使用llhttp库中的wasm_on_message_complete函数来处理请求的消息。

此外，有一个名为wasm_settings的常量，它定义了wasm_on_message_begin、wasm_on_url、wasm_on_status、wasm_on_header_field、wasm_on_header_value、wasm_on_headers_complete、wasm_on_body和wasm_on_message_complete函数的默认实现。如果没有这些函数，这些默认实现将不会被使用。


```cpp
extern int wasm_on_header_value(llhttp_t* p, const char* at, size_t length);
extern int wasm_on_headers_complete(llhttp_t * p);
extern int wasm_on_body(llhttp_t* p, const char* at, size_t length);
extern int wasm_on_message_complete(llhttp_t * p);

const llhttp_settings_t wasm_settings = {
  wasm_on_message_begin,
  wasm_on_url,
  wasm_on_status,
  wasm_on_header_field,
  wasm_on_header_value,
  wasm_on_headers_complete,
  wasm_on_body,
  wasm_on_message_complete,
  NULL,
  NULL,
};


```



这段代码定义了两个函数，一个是要返回一个llhttp_t类型的指针，另一个是释放内存。

llhttp_alloc函数接受一个llhttp_type_t类型的参数，并返回一个llhttp_t类型的指针。函数内部使用llhttp_init函数初始化一个llhttp_t对象，并设置wasm_settings标志位，以便在释放内存时正确地关闭网络连接。

llhttp_free函数接受一个llhttp_t类型的指针，并释放内存。函数内部使用free函数释放内存，并确保在llhttp_free函数返回前释放所有分配的内存。

另外，还定义了两个辅助函数，一个获取解析器的类型，另一个是获取解析器中当前设置的WASM设置。


```cpp
llhttp_t* llhttp_alloc(llhttp_type_t type) {
  llhttp_t* parser = malloc(sizeof(llhttp_t));
  llhttp_init(parser, type, &wasm_settings);
  return parser;
}

void llhttp_free(llhttp_t* parser) {
  free(parser);
}

/* Some getters required to get stuff from the parser */

uint8_t llhttp_get_type(llhttp_t* parser) {
  return parser->type;
}

```

这段代码是一个函数指针，它指向了一个名为"llhttp_get_http_major()"的函数，该函数返回了一个LLHTTP句柄（LLHTTP）的http major值。

接下来，又定义了两个名为"llhttp_get_http_minor()"的函数，它们同样返回了一个LLHTTP句柄的http minor值。

然后，定义了一个名为"llhttp_get_method()"的函数，它返回了一个LLHTTP句柄的method值。

接着，定义了一个名为"llhttp_get_status_code()"的函数，它返回了一个LLHTTP句柄的状态码（status_code）。


```cpp
uint8_t llhttp_get_http_major(llhttp_t* parser) {
  return parser->http_major;
}

uint8_t llhttp_get_http_minor(llhttp_t* parser) {
  return parser->http_minor;
}

uint8_t llhttp_get_method(llhttp_t* parser) {
  return parser->method;
}

int llhttp_get_status_code(llhttp_t* parser) {
  return parser->status_code;
}

```

这两段代码定义了两个名为 `llhttp_get_upgrade` 和 `llhttp_reset` 的函数，以及一个名为 `uint8_t` 的变量 `upgrade`。它们都属于一个名为 `llhttp` 的软件包，其作用是在 Web 应用程序中升级 HTTP 设置。以下是这两段代码的作用解释：

1. `llhttp_get_upgrade(llhttp_t* parser)`函数的作用是获取解析器 `parser` 的 `upgrade` 属性的值，并将其返回。`upgrade` 属性表示 HTTP 设置的升级版本，它的值可以从 0 到 3 中的任意一个。

2. `llhttp_reset(llhttp_t* parser)`函数的作用是重置解析器 `parser` 的状态，包括将其 `type` 设置为 `llhttp_type_t` 类型的初始值 `null`，将其 `settings` 设置为 `null` 指针，将其 `data` 设置为 `null`，将其 `lenient_flags` 设置为 `0`。这个函数在解析器初始化后调用，确保解析器在每次加载数据时都初始化为默认状态。

这两个函数一起工作，确保解析器在每次加载数据时都使用默认的、不包含任何自定义设置的 HTTP 设置。这个软件包通常用于在 Web 应用程序中升级 HTTP 设置，使得开发人员可以使用 Web 开发框架中提供的工具和库。


```cpp
uint8_t llhttp_get_upgrade(llhttp_t* parser) {
  return parser->upgrade;
}

#endif  // defined(__wasm__)


void llhttp_reset(llhttp_t* parser) {
  llhttp_type_t type = parser->type;
  const llhttp_settings_t* settings = parser->settings;
  void* data = parser->data;
  uint8_t lenient_flags = parser->lenient_flags;

  llhttp__internal_init(parser);

  parser->type = type;
  parser->settings = (void*) settings;
  parser->data = data;
  parser->lenient_flags = lenient_flags;
}


```



这段代码定义了两个函数，分别用于设置和完成 HTTP 请求的设置，这两个函数都接受一个指向解析器的指针，以及请求的数据。

llhttp_settings_init函数接受一个指向解析器的 settings 结构体，该结构体包含了请求设置的所有选项，如最大数据大小、连接超时时间等。该函数使用 memset 函数清空 settings 结构体，然后将其长度设置为 0。

llhttp_finish函数用于结束 HTTP 请求，它接收一个指向解析器的指针。该函数首先检查解析器中是否包含错误，如果是，则返回错误码。否则，它将根据设置中包含的选项来决定如何处理请求。具体来说，如果设置中包含了 HTTP_FINISH_SAFE_WITH_CB 选项，则会使用回调函数 on_message_complete 来处理消息，并将结果返回。如果设置中包含了 HTTP_FINISH_SAFE 选项，则不会发生任何错误，直接返回 HTTP_FINISH_OK。如果设置中包含了 HTTP_FINISH_UNSAFE 选项，则会设置解析器的错误码为 "Invalid EOF state"，并返回 HTTP_INVALID_EOF_STATE。如果设置中不包含任何有效的选项，则会直接返回 HTTP_INVALID_EOF_STATE。

注意，在 llhttp_finish 函数中，如果解析器中存在错误，则返回 err，而不是 HPE_INVALID_EOF_STATE。这是因为该函数将在 HTTP 请求完成后调用 on_message_complete 函数，如果设置中包含了 HTTP_FINISH_INVALID_EAHE_WITH_CB 选项，则会在回调函数中执行错误的操作。


```cpp
llhttp_errno_t llhttp_execute(llhttp_t* parser, const char* data, size_t len) {
  return llhttp__internal_execute(parser, data, data + len);
}


void llhttp_settings_init(llhttp_settings_t* settings) {
  memset(settings, 0, sizeof(*settings));
}


llhttp_errno_t llhttp_finish(llhttp_t* parser) {
  int err;

  /* We're in an error state. Don't bother doing anything. */
  if (parser->error != 0) {
    return 0;
  }

  switch (parser->finish) {
    case HTTP_FINISH_SAFE_WITH_CB:
      CALLBACK_MAYBE(parser, on_message_complete, parser);
      if (err != HPE_OK) return err;

    /* FALLTHROUGH */
    case HTTP_FINISH_SAFE:
      return HPE_OK;
    case HTTP_FINISH_UNSAFE:
      parser->reason = "Invalid EOF state";
      return HPE_INVALID_EOF_STATE;
    default:
      abort();
  }
}


```

这两函数是用于处理 HTTP 请求 pause 和 resume 的函数。

当 HTTP 请求在请求过程中遇到暂停（paused）状态时，第一个函数 `llhttp_pause` 被调用，它会将 HTTP 请求的状态设置为 HPE_PAUSED，并返回原因 "Paused"。

当 HTTP 请求在响应过程中遇到暂停（paused）状态时，第二个函数 `llhttp_resume` 被调用，它会将 HTTP 请求的状态设置为 HPE_PAUSED，并返回原因 "Resumed"。

这些函数的主要作用是用于在 HTTP 请求和响应过程中暂停和恢复请求的状态。


```cpp
void llhttp_pause(llhttp_t* parser) {
  if (parser->error != HPE_OK) {
    return;
  }

  parser->error = HPE_PAUSED;
  parser->reason = "Paused";
}


void llhttp_resume(llhttp_t* parser) {
  if (parser->error != HPE_PAUSED) {
    return;
  }

  parser->error = 0;
}


```

这段代码是一个用于 HTTP 解析的库函数，主要作用是在 HTTP 解析器(llhttp_t)升级后进行错误处理。

具体来说，代码中定义了两个函数：

1. `llhttp_resume_after_upgrade()`，该函数的作用是在升级后重新开始解析 HTTP 文档，即使升级过程中出现错误。函数的参数是一个 `llhttp_t` 类型的解析器对象(指向解析器)，返回值是一个 `llhttp_errno_t` 类型的错误码，用于返回当前错误码。

2. `llhttp_get_errno()` 和 `llhttp_get_error_reason()`，这两个函数用于获取解析器对象中的错误信息。函数的第一个参数是一个 `const llhttp_t*` 类型的解析器对象，返回值是错误码(errno)和错误描述(reason)。函数的第二个参数和这两个函数类似，用于获取错误信息和描述。

这些函数的具体实现可能还需要结合具体的服务和库来使用，比如在使用 HTTP 解析器时，可能还需要初始化解析器并处理升级等操作。


```cpp
void llhttp_resume_after_upgrade(llhttp_t* parser) {
  if (parser->error != HPE_PAUSED_UPGRADE) {
    return;
  }

  parser->error = 0;
}


llhttp_errno_t llhttp_get_errno(const llhttp_t* parser) {
  return parser->error;
}


const char* llhttp_get_error_reason(const llhttp_t* parser) {
  return parser->reason;
}


```

这两段代码定义了两个函数，分别用于设置和获取 HTTP 错误信息。

llhttp_set_error_reason(llhttp_t* parser, const char* reason) 的作用是设置给定的 HTTP 错误信息的 reason。这个函数接受一个指向 HTTP 错误信息结构体的指针（parser）和一个字符串（reason）作为参数。设置后，parser->reason指向reason，即parser可以访问到reason中的内容。

llhttp_get_error_pos(const llhttp_t* parser) 的作用是获取给定的 HTTP 错误信息结构体中错误位置的引用。这个函数接受一个指向 HTTP 错误信息结构体的指针（parser）作为参数。它返回parser->error_pos，即可以用来访问错误位置的引用。

llhttp_errno_name(llhttp_errno_t err) 的作用是在定义了一系列 HTTP 错误信息的哈希表，用于根据 HTTP 错误代码获取错误名称。这个函数接受一个 HTTP 错误代码（err）作为参数。它根据哈希表中的 HTTP 错误代码定义错误名称，然后返回该名称。

该代码提供了一个简单的错误处理机制，通过给定的函数，可以方便地设置、获取和输出 HTTP 错误信息，使程序在遇到错误时可以更加轻松地处理错误。


```cpp
void llhttp_set_error_reason(llhttp_t* parser, const char* reason) {
  parser->reason = reason;
}


const char* llhttp_get_error_pos(const llhttp_t* parser) {
  return parser->error_pos;
}


const char* llhttp_errno_name(llhttp_errno_t err) {
#define HTTP_ERRNO_GEN(CODE, NAME, _) case HPE_##NAME: return "HPE_" #NAME;
  switch (err) {
    HTTP_ERRNO_MAP(HTTP_ERRNO_GEN)
    default: abort();
  }
```

这段代码定义了一个名为 HTTP_ERRNO_GEN 的函数，但这个函数并没有定义任何函数体，它只是一个自定义函数声明。这个声明告诉编译器可以在程序中使用 HTTP_ERRNO_GEN 这个函数名。

接下来的代码定义了一个名为 llhttp_method_name 的函数，它接受一个 HTTP 方法类型参数和一个字符串参数。这个函数使用 switch 语句来查找 HTTP 方法名称和相应的函数返回值，然后它将返回函数的名称。函数的实现是将 HTTP_METHOD_MAP(HTTP_METHOD_GEN) 赋值给一个名为 HTTP_METHOD_GEN 的函数，这个函数是一个 macro，它定义了 HTTP_METHOD_GEN(NUM, NAME, STRING) 函数的宏定义。这个 macro 使用 #define 命令定义宏，它将宏名称和参数名称存储在一个 pair 结构中，这个 pair 是 include 和 def宏定义的关键字参数。

接下来是 llhttp_set_lenient_headers 函数，它接受一个指向 llhttp_t 类型的变量和一个布尔值参数。如果布尔值参数为真，那么它将设置 parser 的 lenient_headers 成员变量为真，这意味着 parser 将解析 HTTP 头部，并且不会捕获任何 HTTP 错误。如果布尔值参数为假，那么它将设置 parser 的 lenient_headers 成员变量为 false，这意味着 parser 将丢弃 HTTP 头部，并且尝试捕获所有 HTTP 错误。

最后，printf() 函数用于输出 HTTP 状态码。


```cpp
#undef HTTP_ERRNO_GEN
}


const char* llhttp_method_name(llhttp_method_t method) {
#define HTTP_METHOD_GEN(NUM, NAME, STRING) case HTTP_##NAME: return #STRING;
  switch (method) {
    HTTP_METHOD_MAP(HTTP_METHOD_GEN)
    default: abort();
  }
#undef HTTP_METHOD_GEN
}


void llhttp_set_lenient_headers(llhttp_t* parser, int enabled) {
  if (enabled) {
    parser->lenient_flags |= LENIENT_HEADERS;
  } else {
    parser->lenient_flags &= ~LENIENT_HEADERS;
  }
}


```

这两函数函数是设置LLHTTP解析器的一些相关设置，主要作用是控制LLHTTP解析器的行为。

`llhttp_set_lenient_chunked_length`函数的作用是设置LLHTTP解析器是否在解析数据时使用分块的数据。如果`enabled`参数为1，那么设置为启用分块的数据；如果为0，那么禁用分块的数据。

`llhttp_set_lenient_keep_alive`函数的作用是设置LLHTTP解析器是否在发送数据时保持存活（即保持连接）。如果`enabled`参数为1，那么设置为启用保持存活；如果为0，那么禁用保持存活。

这两个函数函数可以用来设置LLHTTP解析器的行为，这对于使用LLHTTP进行网络请求的使用非常有帮助。


```cpp
void llhttp_set_lenient_chunked_length(llhttp_t* parser, int enabled) {
  if (enabled) {
    parser->lenient_flags |= LENIENT_CHUNKED_LENGTH;
  } else {
    parser->lenient_flags &= ~LENIENT_CHUNKED_LENGTH;
  }
}


void llhttp_set_lenient_keep_alive(llhttp_t* parser, int enabled) {
  if (enabled) {
    parser->lenient_flags |= LENIENT_KEEP_ALIVE;
  } else {
    parser->lenient_flags &= ~LENIENT_KEEP_ALIVE;
  }
}

```

这段代码定义了两个回调函数，用于处理HTTP请求中的URL部分。

第一个回调函数是`llhttp__on_url`，它接受一个LLHTTP句柄（LLHTTP_T结构体），一个URL参数（const char* 类型的指针），以及一个结束URL参数（const char* 类型的指针）。这个函数的作用是在LLHTTP句柄中设置请求的URL，然后返回设置的err码。

第二个回调函数是`llhttp__on_message_begin`，它也接受一个LLHTTP句柄和一个URL参数，但它的作用是在LLHTTP句柄中设置URL，然后返回设置的err码。这个函数的实现与`llhttp__on_url`非常类似，只是设置的参数中少了结束URL参数。

这两个回调函数的实现主要依赖于LLHTTP库，它提供了一系列用于处理HTTP请求的功能，包括`llhttp_init`、`llhttp_send`、`llhttp_receive`等等。通过使用这些函数，可以方便地设置请求的URL，完成HTTP请求的工作。


```cpp
/* Callbacks */


int llhttp__on_message_begin(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_message_begin, s);
  return err;
}


int llhttp__on_url(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_url, s, p, endp - p);
  return err;
}


```



这段代码定义了三个名为"llhttp__on_url_complete"、"llhttp__on_status"和"llhttp__on_status_complete"的函数，它们都是llhttp_t类型的函数指针。

这些函数接受一个llhttp_t类型的上下文对象(llhttp_t* s)，以及一个指向字符串的指针(const char* p)和一个指向字符串结束的指针(const char* endp)，然后返回一个int类型的错误码。

这三个函数的作用是处理llhttp_t对象在请求URL时的状态更新。当请求URL成功完成时，on_url_complete函数会被调用，并传入llhttp_t对象和起始URL和结束URL。函数返回0，表示请求成功。

当请求URL的状态发生变化时，on_status函数会被调用。on_status函数需要传递两个参数，一个是llhttp_t对象，另一个是起始URL和结束URL。函数返回0，表示状态未发生变化。

当请求URL的状态完成更改时，on_status_complete函数会被调用。on_status_complete函数需要传递一个llhttp_t对象和一个指向字符串结束的指针。函数返回0，表示状态已完成更改。


```cpp
int llhttp__on_url_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_url_complete, s);
  return err;
}


int llhttp__on_status(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_status, s, p, endp - p);
  return err;
}


int llhttp__on_status_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_status_complete, s);
  return err;
}


```

这段代码定义了三个名为"llhttp__on_header_field"、"llhttp__on_header_field_complete"和"llhttp__on_header_value"的函数，它们都是llhttp_t类型的函数指针。

这些函数的作用是处理HTTP请求头中的字段。其中，llhttp__on_header_field函数用于处理在请求头中找到的指定字段，llhttp__on_header_field_complete函数用于处理在请求头中找到的指定字段的结束，而llhttp__on_header_value函数用于处理在请求头中找到的指定字段。

每个函数都采用了一个callback函数作为其实现方式，该函数需要传递两个参数，一个是输入llhttp_t类型的句柄(s)，另一个是请求头中的字段名称(p)和请求头中的结束字段名称(endp)的指针。这些函数的实现都使用了一个无返回值的函数指针，因此无法从函数中返回任何值。


```cpp
int llhttp__on_header_field(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_header_field, s, p, endp - p);
  return err;
}


int llhttp__on_header_field_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_header_field_complete, s);
  return err;
}


int llhttp__on_header_value(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_header_value, s, p, endp - p);
  return err;
}


```

这三段代码都是LLHTTP框架中的回调函数，用于处理HTTP请求中的不同阶段。它们的具体作用如下：

1. int llhttp__on_header_value_complete(llhttp_t* s, const char* p, const char* endp) {
  这段代码定义了一个名为`llhttp__on_header_value_complete`的函数，它接收一个LLHTTP句柄（LLHTTP_T）、一个头部名称（const char*）和一个结束头部名称（const char*）作为参数。它返回一个整数表示错误码，如果有错误发生则返回，否则函数返回0。

  这个函数的作用是在LLHTTP框架中，当处理到HTTP请求的头部阶段时，设置LLHTTP句柄的`on_header_value_complete`回调函数的输入参数，并将`p`和`endp`作为输入参数传入。

2. int llhttp__on_headers_complete(llhttp_t* s, const char* p, const char* endp) {
  这段代码定义了一个名为`llhttp__on_headers_complete`的函数，它与`llhttp__on_header_value_complete`类似，只是返回类型不同。它接收一个LLHTTP句柄（LLHTTP_T）、一个头部名称（const char*）和一个结束头部名称（const char*）作为输入参数。它返回一个整数表示错误码，如果有错误发生则返回，否则函数返回0。

  这个函数的作用与`llhttp__on_header_value_complete`函数类似，只是返回类型不同。

3. int llhttp__on_message_complete(llhttp_t* s, const char* p, const char* endp) {
  这段代码定义了一个名为`llhttp__on_message_complete`的函数，它与前两个函数类似，只是输入参数的类型不同。它接收一个LLHTTP句柄（LLHTTP_T）、一个头部名称（const char*）和一个结束头部名称（const char*）作为输入参数。它返回一个整数表示错误码，如果有错误发生则返回，否则函数返回0。

  这个函数的作用与`llhttp__on_header_value_complete`和`llhttp__on_headers_complete`函数类似，只是输入参数的类型不同。


```cpp
int llhttp__on_header_value_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_header_value_complete, s);
  return err;
}


int llhttp__on_headers_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_headers_complete, s);
  return err;
}


int llhttp__on_message_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_message_complete, s);
  return err;
}


```

这段代码定义了三个名为"llhttp__on_body"、"llhttp__on_chunk_header"和"llhttp__on_chunk_complete"的函数，它们都是llhttp_t类型的函数指针。

这三个函数的作用是：

1. "llhttp__on_body"函数，它接收三个参数：一个指向llhttp_t对象的指针s，一个指向字符串p的指针，和一个指向字符串endp的指针。它的返回值是一个int类型的错误码，用于指示在什么地方出现错误。

2. "llhttp__on_chunk_header"函数，它与"llhttp__on_body"函数相似，只是参数p和endp的顺序颠倒了，它接收三个参数：一个指向llhttp_t对象的指针s和一个指向字符串p的指针，以及一个指向字符串endp的指针。它的返回值是一个int类型的错误码，用于指示在什么地方出现错误。

3. "llhttp__on_chunk_complete"函数，它与"llhttp__on_chunk_header"函数相似，只是参数p和endp的顺序颠倒了，它接收三个参数：一个指向llhttp_t对象的指针s和一个指向字符串p的指针，以及一个指向字符串endp的指针。它的返回值是一个int类型的错误码，用于指示在什么地方出现错误。

这三个函数的具体实现可能因应用程序的需求而异。


```cpp
int llhttp__on_body(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_body, s, p, endp - p);
  return err;
}


int llhttp__on_chunk_header(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_chunk_header, s);
  return err;
}


int llhttp__on_chunk_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_chunk_complete, s);
  return err;
}


```

这段代码定义了一个名为`llhttp__debug`的函数，其作用是打印LLHTTP库中的调试信息。

具体来说，该函数接受三个参数：

- `s`：指向LLHTTP对象的指针，包含LLHTTP类型、 flags和 Next指针等信息。
- `p`：当前正在处理的字符串，包含了LLHTTP对象从客户端收到的数据。
- `endp`：表示当前字符串的结束位置，也就是LLHTTP对象中所有数据的结束位置。
- `msg`：当前正在打印的调试信息的字符串。

函数的实现采用了if语句，如果当前正在处理的字符串与结束字符串相等，就打印一条调试信息并返回。否则，打印一条调试信息，其中当前正在处理的字符串、LLHTTP类型和flags都会被打印出来，然后是下一个LLHTTP对象的Next指针，然后是调试信息的字符串。


```cpp
/* Private */


void llhttp__debug(llhttp_t* s, const char* p, const char* endp,
                   const char* msg) {
  if (p == endp) {
    fprintf(stderr, "p=%p type=%d flags=%02x next=null debug=%s\n", s, s->type,
            s->flags, msg);
  } else {
    fprintf(stderr, "p=%p type=%d flags=%02x next=%02x   debug=%s\n", s,
            s->type, s->flags, *p, msg);
  }
}

```

# `src/3rdparty/llhttp/http.c`

这段代码是一个 C 语言的程序，定义了两个名为 "llhttp_message_needs_eof" 和 "llhttp_should_keep_alive" 的函数，以及一个名为 "llhttp__before_headers_complete" 的函数。它们都在 Linux HTTP 客户端库 "llhttp" 中被使用。

这段代码的作用是设置 HTTP 客户端在发送 HTTP 请求或响应前，解析 HTTP 头部的 Content-Length 和 Set-Cookie 标头中的 Host 是否发生变化，从而判断是否需要继续发送数据。

具体来说，`llhttp_message_needs_eof` 函数用于在客户端发送 HTTP 请求或响应时，判断是否需要发送 Content-Length 和 Set-Cookie 标头。如果客户端已经发送了 Content-Length 和 Set-Cookie 标头，并且当前版本号为 1.1，则不需要发送 Content-Length 标头，否则需要发送。

`llhttp_should_keep_alive` 函数用于在客户端发送 HTTP 请求或响应时，判断是否需要保持与服务器的连接。如果客户端已经发送了 Set-Cookie 标头，并且当前版本号为 1.1，则不需要保持与服务器的连接，否则需要保持。

`llhttp__before_headers_complete` 函数用于在客户端发送 HTTP 请求或响应前，设置 Content-Length 和 Set-Cookie 标头。这里使用了 `if` 和 `else` 语句，当客户端已经发送了 Content-Length 和 Set-Cookie 标头，并且当前版本号为 1.1 时，设置 Content-Length，否则设置 Set-Cookie 标头。


```cpp
#include <stdio.h>
#ifndef LLHTTP__TEST
# include "llhttp.h"
#else
# define llhttp_t llparse_t
#endif  /* */

int llhttp_message_needs_eof(const llhttp_t* parser);
int llhttp_should_keep_alive(const llhttp_t* parser);

int llhttp__before_headers_complete(llhttp_t* parser, const char* p,
                                    const char* endp) {
  /* Set this here so that on_headers_complete() callbacks can see it */
  if ((parser->flags & F_UPGRADE) &&
      (parser->flags & F_CONNECTION_UPGRADE)) {
    /* For responses, "Upgrade: foo" and "Connection: upgrade" are
     * mandatory only when it is a 101 Switching Protocols response,
     * otherwise it is purely informational, to announce support.
     */
    parser->upgrade =
        (parser->type == HTTP_REQUEST || parser->status_code == 101);
  } else {
    parser->upgrade = (parser->method == HTTP_CONNECT);
  }
  return 0;
}


```

This is a function written in the Go programming language that parses the body of an HTTP request message. It returns one of the following values:

* 0: The message body has the wrong format.
* 1: The Content-Length header is missing.
* 2: The message body is chunked.
* 3: The Content-Length header is present and the message body is not a complete chunk.
* 4: The message body has the correct format, but the Content-Length header is not specific to the chunked transfer encoding.
* 5: The message body has the correct format, but the server must respond with a 400 HTTP status code.

The function also takes note of the `F_SKIPBODY` flag, which indicates that the body of the message should be ignored.

I hope this helps! Let me know if you have any other questions.


```cpp
/* Return values:
 * 0 - No body, `restart`, message_complete
 * 1 - CONNECT request, `restart`, message_complete, and pause
 * 2 - chunk_size_start
 * 3 - body_identity
 * 4 - body_identity_eof
 * 5 - invalid transfer-encoding for request
 */
int llhttp__after_headers_complete(llhttp_t* parser, const char* p,
                                   const char* endp) {
  int hasBody;

  hasBody = parser->flags & F_CHUNKED || parser->content_length > 0;
  if (parser->upgrade && (parser->method == HTTP_CONNECT ||
                          (parser->flags & F_SKIPBODY) || !hasBody)) {
    /* Exit, the rest of the message is in a different protocol. */
    return 1;
  }

  if (parser->flags & F_SKIPBODY) {
    return 0;
  } else if (parser->flags & F_CHUNKED) {
    /* chunked encoding - ignore Content-Length header, prepare for a chunk */
    return 2;
  } else if (parser->flags & F_TRANSFER_ENCODING) {
    if (parser->type == HTTP_REQUEST &&
        (parser->lenient_flags & LENIENT_CHUNKED_LENGTH) == 0) {
      /* RFC 7230 3.3.3 */

      /* If a Transfer-Encoding header field
       * is present in a request and the chunked transfer coding is not
       * the final encoding, the message body length cannot be determined
       * reliably; the server MUST respond with the 400 (Bad Request)
       * status code and then close the connection.
       */
      return 5;
    } else {
      /* RFC 7230 3.3.3 */

      /* If a Transfer-Encoding header field is present in a response and
       * the chunked transfer coding is not the final encoding, the
       * message body length is determined by reading the connection until
       * it is closed by the server.
       */
      return 4;
    }
  } else {
    if (!(parser->flags & F_CONTENT_LENGTH)) {
      if (!llhttp_message_needs_eof(parser)) {
        /* Assume content-length 0 - read the next */
        return 0;
      } else {
        /* Read body until EOF */
        return 4;
      }
    } else if (parser->content_length == 0) {
      /* Content-Length header given but zero: Content-Length: 0\r\n */
      return 0;
    } else {
      /* Content-Length header given and non-zero */
      return 3;
    }
  }
}


```

这两段代码定义了两个函数，名为`llhttp__after_message_complete`和`llhttp_message_needs_eof`。它们的作用是确保解析器在处理HTTP请求或响应时能够正确地完成操作。

`llhttp__after_message_complete`函数接收一个`llhttp_t`类型的解析器和一个字符串指针`p`，以及一个字符串指针`endp`。它的主要作用是确保解析器在完成解析请求或响应时能够继续工作，即使解析器已经在处理请求或响应的头部或数据部分。为此，函数首先检查`llhttp_should_keep_alive`函数的值，然后将`parser->finish`设置为`HTTP_FINISH_SAFE`，将`parser->flags`设置为0。这些设置确保解析器在完成解析请求或响应时不会被阻塞，并且它也不会继续尝试解析尚未完成的部分。

`llhttp_message_needs_eof`函数与`llhttp__after_message_complete`函数类似，但它的检查点在于解析器是否需要输出HTTP响应的结尾。这个函数主要检查解析器是否处于宽松解析模式（使用`llhttp_should_keep_alive`函数时）以及是否已经解析完了所有的数据。如果是宽松解析模式，则该函数将返回`0`，表示解析器已经准备好处理任何后续的请求或响应，可以继续执行。否则，如果解析器仍然解析请求或响应的头部或数据部分，该函数将返回`1`，以通知解析器有后续请求或响应尚未完成。


```cpp
int llhttp__after_message_complete(llhttp_t* parser, const char* p,
                                   const char* endp) {
  int should_keep_alive;

  should_keep_alive = llhttp_should_keep_alive(parser);
  parser->finish = HTTP_FINISH_SAFE;
  parser->flags = 0;

  /* NOTE: this is ignored in loose parsing mode */
  return should_keep_alive;
}


int llhttp_message_needs_eof(const llhttp_t* parser) {
  if (parser->type == HTTP_REQUEST) {
    return 0;
  }

  /* See RFC 2616 section 4.4 */
  if (parser->status_code / 100 == 1 || /* 1xx e.g. Continue */
      parser->status_code == 204 ||     /* No Content */
      parser->status_code == 304 ||     /* Not Modified */
      (parser->flags & F_SKIPBODY)) {     /* response to a HEAD request */
    return 0;
  }

  /* RFC 7230 3.3.3, see `llhttp__after_headers_complete` */
  if ((parser->flags & F_TRANSFER_ENCODING) &&
      (parser->flags & F_CHUNKED) == 0) {
    return 1;
  }

  if (parser->flags & (F_CHUNKED | F_CONTENT_LENGTH)) {
    return 0;
  }

  return 1;
}


```

这段代码是一个名为 `llhttp_should_keep_alive` 的函数，它是用来判断 HTTP 连接是否需要保持连接的。该函数的作用是，如果解析器(llhttp_t)的 HTTP  major 和 HTTP minor 都大于 0，那么就执行以下操作：

1. 如果解析器开启了连接关闭(F_CONNECTION_CLOSE)标志，那么返回 0，表示不需要保持连接。
2. 如果解析器开启了连接保持存活(F_CONNECTION_KEEP_ALIVE)标志，那么需要检查是否需要发送 EOF 消息，并且返回 0，表示不需要保持连接。
3. 如果解析器开启了连接降级(F_CONNECTION_KEEP_ALIVE)标志，那么不会发送 EOF 消息，并且返回 1，表示需要保持连接。

换句话说，该函数会根据 HTTP 协议版本和连接标志来判断是否需要保持连接，并且返回一个指示符表示结果。


```cpp
int llhttp_should_keep_alive(const llhttp_t* parser) {
  if (parser->http_major > 0 && parser->http_minor > 0) {
    /* HTTP/1.1 */
    if (parser->flags & F_CONNECTION_CLOSE) {
      return 0;
    }
  } else {
    /* HTTP/1.0 or earlier */
    if (!(parser->flags & F_CONNECTION_KEEP_ALIVE)) {
      return 0;
    }
  }

  return !llhttp_message_needs_eof(parser);
}

```

# llhttp
[![CI](https://github.com/nodejs/llhttp/workflows/CI/badge.svg)](https://github.com/nodejs/llhttp/actions?query=workflow%3ACI)

Port of [http_parser][0] to [llparse][1].

## Why?

Let's face it, [http_parser][0] is practically unmaintainable. Even
introduction of a single new method results in a significant code churn.

This project aims to:

* Make it maintainable
* Verifiable
* Improving benchmarks where possible

More details in [Fedor Indutny's talk at JSConf EU 2019](https://youtu.be/x3k_5Mi66sY)

## How?

Over time, different approaches for improving [http_parser][0]'s code base
were tried. However, all of them failed due to resulting significant performance
degradation.

This project is a port of [http_parser][0] to TypeScript. [llparse][1] is used
to generate the output C source file, which could be compiled and
linked with the embedder's program (like [Node.js][7]).

## Performance

So far llhttp outperforms http_parser:

|                 | input size |  bandwidth   |  reqs/sec  |   time  |
|:----------------|-----------:|-------------:|-----------:|--------:|
| **llhttp**      | 8192.00 mb | 1777.24 mb/s | 3583799.39 req/sec | 4.61 s |
| **http_parser** | 8192.00 mb | 694.66 mb/s | 1406180.33 req/sec | 11.79 s |

llhttp is faster by approximately **156%**.

## Maintenance

llhttp project has about 1400 lines of TypeScript code describing the parser
itself and around 450 lines of C code and headers providing the helper methods.
The whole [http_parser][0] is implemented in approximately 2500 lines of C, and
436 lines of headers.

All optimizations and multi-character matching in llhttp are generated
automatically, and thus doesn't add any extra maintenance cost. On the contrary,
most of http_parser's code is hand-optimized and unrolled. Instead describing
"how" it should parse the HTTP requests/responses, a maintainer should
implement the new features in [http_parser][0] cautiously, considering
possible performance degradation and manually optimizing the new code.

## Verification

The state machine graph is encoded explicitly in llhttp. The [llparse][1]
automatically checks the graph for absence of loops and correct reporting of the
input ranges (spans) like header names and values. In the future, additional
checks could be performed to get even stricter verification of the llhttp.

## Usage

```cppC
#include "llhttp.h"

llhttp_t parser;
llhttp_settings_t settings;

/* Initialize user callbacks and settings */
llhttp_settings_init(&settings);

/* Set user callback */
settings.on_message_complete = handle_on_message_complete;

/* Initialize the parser in HTTP_BOTH mode, meaning that it will select between
 * HTTP_REQUEST and HTTP_RESPONSE parsing automatically while reading the first
 * input.
 */
llhttp_init(&parser, HTTP_BOTH, &settings);

/* Parse request! */
const char* request = "GET / HTTP/1.1\r\n\r\n";
int request_len = strlen(request);

enum llhttp_errno err = llhttp_execute(&parser, request, request_len);
if (err == HPE_OK) {
  /* Successfully parsed! */
} else {
  fprintf(stderr, "Parse error: %s %s\n", llhttp_errno_name(err),
          parser.reason);
}
```

---

### Bindings to other languages

* Python: [pallas/pyllhttp][8]
* Ruby: [metabahn/llhttp][9]

#### LICENSE

This software is licensed under the MIT License.

Copyright Fedor Indutny, 2018.

Permission is hereby granted, free of charge, to any person obtaining a
copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to permit
persons to whom the Software is furnished to do so, subject to the
following conditions:

The above copyright notice and this permission notice shall be included
in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN
NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR
OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE
USE OR OTHER DEALINGS IN THE SOFTWARE.

[0]: https://github.com/nodejs/http-parser
[1]: https://github.com/nodejs/llparse
[2]: https://en.wikipedia.org/wiki/Register_allocation#Spilling
[3]: https://en.wikipedia.org/wiki/Tail_call
[4]: https://llvm.org/docs/LangRef.html
[5]: https://llvm.org/docs/LangRef.html#call-instruction
[6]: https://clang.llvm.org/
[7]: https://github.com/nodejs/node
[8]: https://github.com/pallas/pyllhttp
[9]: https://github.com/metabahn/llhttp


![RapidJSON logo](doc/logo/rapidjson.png)

![Release version](https://img.shields.io/badge/release-v1.1.0-blue.svg)

## A fast JSON parser/generator for C++ with both SAX/DOM style API

Tencent is pleased to support the open source community by making RapidJSON available.

Copyright (C) 2015 THL A29 Limited, a Tencent company, and Milo Yip.

* [RapidJSON GitHub](https://github.com/Tencent/rapidjson/)
* RapidJSON Documentation
  * [English](http://rapidjson.org/)
  * [简体中文](http://rapidjson.org/zh-cn/)
  * [GitBook](https://www.gitbook.com/book/miloyip/rapidjson/) with downloadable PDF/EPUB/MOBI, without API reference.

## Build status

| [Linux][lin-link] | [Windows][win-link] | [Coveralls][cov-link] |
| :---------------: | :-----------------: | :-------------------: |
| ![lin-badge]      | ![win-badge]        | ![cov-badge]          |

[lin-badge]: https://travis-ci.org/Tencent/rapidjson.svg?branch=master "Travis build status"
[lin-link]:  https://travis-ci.org/Tencent/rapidjson "Travis build status"
[win-badge]: https://ci.appveyor.com/api/projects/status/l6qulgqahcayidrf/branch/master?svg=true "AppVeyor build status"
[win-link]:  https://ci.appveyor.com/project/miloyip/rapidjson-0fdqj/branch/master "AppVeyor build status"
[cov-badge]: https://coveralls.io/repos/Tencent/rapidjson/badge.svg?branch=master "Coveralls coverage"
[cov-link]:  https://coveralls.io/r/Tencent/rapidjson?branch=master "Coveralls coverage"

## Introduction

RapidJSON is a JSON parser and generator for C++. It was inspired by [RapidXml](http://rapidxml.sourceforge.net/).

* RapidJSON is **small** but **complete**. It supports both SAX and DOM style API. The SAX parser is only a half thousand lines of code.

* RapidJSON is **fast**. Its performance can be comparable to `strlen()`. It also optionally supports SSE2/SSE4.2 for acceleration.

* RapidJSON is **self-contained** and **header-only**. It does not depend on external libraries such as BOOST. It even does not depend on STL.

* RapidJSON is **memory-friendly**. Each JSON value occupies exactly 16 bytes for most 32/64-bit machines (excluding text string). By default it uses a fast memory allocator, and the parser allocates memory compactly during parsing.

* RapidJSON is **Unicode-friendly**. It supports UTF-8, UTF-16, UTF-32 (LE & BE), and their detection, validation and transcoding internally. For example, you can read a UTF-8 file and let RapidJSON transcode the JSON strings into UTF-16 in the DOM. It also supports surrogates and "\u0000" (null character).

More features can be read [here](doc/features.md).

JSON(JavaScript Object Notation) is a light-weight data exchange format. RapidJSON should be in full compliance with RFC7159/ECMA-404, with optional support of relaxed syntax. More information about JSON can be obtained at
* [Introducing JSON](http://json.org/)
* [RFC7159: The JavaScript Object Notation (JSON) Data Interchange Format](https://tools.ietf.org/html/rfc7159)
* [Standard ECMA-404: The JSON Data Interchange Format](https://www.ecma-international.org/publications/standards/Ecma-404.htm)

## Highlights in v1.1 (2016-8-25)

* Added [JSON Pointer](doc/pointer.md)
* Added [JSON Schema](doc/schema.md)
* Added [relaxed JSON syntax](doc/dom.md) (comment, trailing comma, NaN/Infinity)
* Iterating array/object with [C++11 Range-based for loop](doc/tutorial.md)
* Reduce memory overhead of each `Value` from 24 bytes to 16 bytes in x86-64 architecture.

For other changes please refer to [change log](CHANGELOG.md).

## Compatibility

RapidJSON is cross-platform. Some platform/compiler combinations which have been tested are shown as follows.
* Visual C++ 2008/2010/2013 on Windows (32/64-bit)
* GNU C++ 3.8.x on Cygwin
* Clang 3.4 on Mac OS X (32/64-bit) and iOS
* Clang 3.4 on Android NDK

Users can build and run the unit tests on their platform/compiler.

## Installation

RapidJSON is a header-only C++ library. Just copy the `include/rapidjson` folder to system or project's include path.

Alternatively, if you are using the [vcpkg](https://github.com/Microsoft/vcpkg/) dependency manager you can download and install rapidjson with CMake integration in a single command:
* vcpkg install rapidjson

RapidJSON uses following software as its dependencies:
* [CMake](https://cmake.org/) as a general build tool
* (optional) [Doxygen](http://www.doxygen.org) to build documentation
* (optional) [googletest](https://github.com/google/googletest) for unit and performance testing

To generate user documentation and run tests please proceed with the steps below:

1. Execute `git submodule update --init` to get the files of thirdparty submodules (google test).
2. Create directory called `build` in rapidjson source directory.
3. Change to `build` directory and run `cmake ..` command to configure your build. Windows users can do the same with cmake-gui application.
4. On Windows, build the solution found in the build directory. On Linux, run `make` from the build directory.

On successful build you will find compiled test and example binaries in `bin`
directory. The generated documentation will be available in `doc/html`
directory of the build tree. To run tests after finished build please run `make
test` or `ctest` from your build tree. You can get detailed output using `ctest
-V` command.

It is possible to install library system-wide by running `make install` command
from the build tree with administrative privileges. This will install all files
according to system preferences.  Once RapidJSON is installed, it is possible
to use it from other CMake projects by adding `find_package(RapidJSON)` line to
your CMakeLists.txt.

## Usage at a glance

This simple example parses a JSON string into a document (DOM), make a simple modification of the DOM, and finally stringify the DOM to a JSON string.

~~~~~~~~~~cpp
// rapidjson/example/simpledom/simpledom.cpp`
#include "rapidjson/document.h"
#include "rapidjson/writer.h"
#include "rapidjson/stringbuffer.h"
#include <iostream>

using namespace rapidjson;

int main() {
    // 1. Parse a JSON string into DOM.
    const char* json = "{\"project\":\"rapidjson\",\"stars\":10}";
    Document d;
    d.Parse(json);

    // 2. Modify it by DOM.
    Value& s = d["stars"];
    s.SetInt(s.GetInt() + 1);

    // 3. Stringify the DOM
    StringBuffer buffer;
    Writer<StringBuffer> writer(buffer);
    d.Accept(writer);

    // Output {"project":"rapidjson","stars":11}
    std::cout << buffer.GetString() << std::endl;
    return 0;
}
~~~~~~~~~~

Note that this example did not handle potential errors.

The following diagram shows the process.

![simpledom](doc/diagram/simpledom.png)

More [examples](https://github.com/Tencent/rapidjson/tree/master/example) are available:

* DOM API
  * [tutorial](https://github.com/Tencent/rapidjson/blob/master/example/tutorial/tutorial.cpp): Basic usage of DOM API.

* SAX API
  * [simplereader](https://github.com/Tencent/rapidjson/blob/master/example/simplereader/simplereader.cpp): Dumps all SAX events while parsing a JSON by `Reader`.
  * [condense](https://github.com/Tencent/rapidjson/blob/master/example/condense/condense.cpp): A command line tool to rewrite a JSON, with all whitespaces removed.
  * [pretty](https://github.com/Tencent/rapidjson/blob/master/example/pretty/pretty.cpp): A command line tool to rewrite a JSON with indents and newlines by `PrettyWriter`.
  * [capitalize](https://github.com/Tencent/rapidjson/blob/master/example/capitalize/capitalize.cpp): A command line tool to capitalize strings in JSON.
  * [messagereader](https://github.com/Tencent/rapidjson/blob/master/example/messagereader/messagereader.cpp): Parse a JSON message with SAX API.
  * [serialize](https://github.com/Tencent/rapidjson/blob/master/example/serialize/serialize.cpp): Serialize a C++ object into JSON with SAX API.
  * [jsonx](https://github.com/Tencent/rapidjson/blob/master/example/jsonx/jsonx.cpp): Implements a `JsonxWriter` which stringify SAX events into [JSONx](https://www-01.ibm.com/support/knowledgecenter/SS9H2Y_7.1.0/com.ibm.dp.doc/json_jsonx.html) (a kind of XML) format. The example is a command line tool which converts input JSON into JSONx format.

* Schema
  * [schemavalidator](https://github.com/Tencent/rapidjson/blob/master/example/schemavalidator/schemavalidator.cpp) : A command line tool to validate a JSON with a JSON schema.

* Advanced
  * [prettyauto](https://github.com/Tencent/rapidjson/blob/master/example/prettyauto/prettyauto.cpp): A modified version of [pretty](https://github.com/Tencent/rapidjson/blob/master/example/pretty/pretty.cpp) to automatically handle JSON with any UTF encodings.
  * [parsebyparts](https://github.com/Tencent/rapidjson/blob/master/example/parsebyparts/parsebyparts.cpp): Implements an `AsyncDocumentParser` which can parse JSON in parts, using C++11 thread.
  * [filterkey](https://github.com/Tencent/rapidjson/blob/master/example/filterkey/filterkey.cpp): A command line tool to remove all values with user-specified key.
  * [filterkeydom](https://github.com/Tencent/rapidjson/blob/master/example/filterkeydom/filterkeydom.cpp): Same tool as above, but it demonstrates how to use a generator to populate a `Document`.

## Contributing

RapidJSON welcomes contributions. When contributing, please follow the code below.

### Issues

Feel free to submit issues and enhancement requests.

Please help us by providing **minimal reproducible examples**, because source code is easier to let other people understand what happens.
For crash problems on certain platforms, please bring stack dump content with the detail of the OS, compiler, etc.

Please try breakpoint debugging first, tell us what you found, see if we can start exploring based on more information been prepared.

### Workflow

In general, we follow the "fork-and-pull" Git workflow.

 1. **Fork** the repo on GitHub
 2. **Clone** the project to your own machine
 3. **Checkout** a new branch on your fork, start developing on the branch
 4. **Test** the change before commit, Make sure the changes pass all the tests, including `unittest` and `preftest`, please add test case for each new feature or bug-fix if needed.
 5. **Commit** changes to your own branch
 6. **Push** your work back up to your fork
 7. Submit a **Pull request** so that we can review your changes

NOTE: Be sure to merge the latest from "upstream" before making a pull request!

### Copyright and Licensing

You can copy and paste the license summary from below.

```cpp
Tencent is pleased to support the open source community by making RapidJSON available.

Copyright (C) 2015 THL A29 Limited, a Tencent company, and Milo Yip.

Licensed under the MIT License (the "License"); you may not use this file except
in compliance with the License. You may obtain a copy of the License at

http://opensource.org/licenses/MIT

Unless required by applicable law or agreed to in writing, software distributed
under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.
```


# GhostRider (Raptoreum) release notes

**XMRig** supports GhostRider algorithm starting from version **v6.16.0**.

No tuning is required - auto-config works well on most CPUs!

### Sample command line (non-SSL port)
```cpp
xmrig -a gr -o raptoreumemporium.com:3008 -u WALLET_ADDRESS -p x
```

### Sample command line (SSL port)
```cpp
xmrig -a gr -o rtm.suprnova.cc:4273 --tls -u WALLET_ADDRESS -p x
```

You can use **rtm_ghostrider_example.cmd** as a template and put pool URL and your wallet address there. The general XMRig documentation is available [here](https://xmrig.com/docs/miner).

**Using `--threads` or `-t` option is NOT recommended because it turns off advanced built-in config.** If you want to tweak the nubmer of threads used for GhostRider, it's recommended to start using config.json instead of command line. The best suitable command line option for this is `--cpu-max-threads-hint=N` where N can be between 0 and 100.

## Performance

While individual algorithm implementations are a bit unoptimized, XMRig achieves higher hashrates by employing better auto-config and more fine-grained thread scheduling: it can calculate a single batch of hashes using 2 threads for parts that don't require much cache. For example, on a typical Intel CPU (2 MB cache per core) it will use 1 thread per core for cn/fast, and 2 threads per core for other Cryptonight variants while calculating the same batch of hashes, always achieving more than 50% CPU load.

For the same reason, XMRig can sometimes use less than 100% CPU on Ryzen 3000/5000 CPUs if it finds that running 1 thread per core is faster for some Cryptonight variants on your system.

**Windows** (detailed results [here](https://imgur.com/a/0njIVVW))
CPU|cpuminer-gr-avx2 1.2.4.1 (tuned), h/s|XMRig v6.16.2 (MSVC build), h/s|Speedup
-|-|-|-
AMD Ryzen 7 4700U|632.6|733.1|+15.89%
Intel Core i7-2600|496.4|554.6|+11.72%
AMD Ryzen 7 3700X @ 4.1 GHz|2453.0|2496.5|+1.77%
AMD Ryzen 5 5600X @ 4.65 GHz|2112.6|2337.5|+10.65%

**Linux (outdated)** (tested by **Delgon**, detailed results [here](https://cdn.discordapp.com/attachments/604375870236524574/913167614749048872/unknown.png))
CPU|cpuminer-gr-avx2 1.2.4.1 (tuned), h/s|XMRig v6.16.0 (GCC build), h/s|Speedup
-|-|-|-
AMD Ryzen 9 3900X|3746.51|3604.89|-3.78%
2xIntel Xeon E5-2698v3|2563.4|2638.38|+2.925%
