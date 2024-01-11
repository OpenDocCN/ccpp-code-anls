# `xmrig\src\3rdparty\hwloc\src\topology-xml-nolibxml.c`

```
/*
 * 版权声明
 */
#include "private/autogen/config.h"  // 包含自动生成的配置文件
#include "hwloc.h"  // 包含硬件拓扑库的头文件
#include "hwloc/plugins.h"  // 包含硬件拓扑库的插件头文件
#include "private/private.h"  // 包含私有的头文件
#include "private/misc.h"  // 包含私有的杂项头文件
#include "private/xml.h"  // 包含私有的 XML 头文件
#include "private/debug.h"  // 包含私有的调试头文件

#include <string.h>  // 包含字符串操作的头文件
#include <assert.h>  // 包含断言的头文件
#include <sys/types.h>  // 包含系统类型的头文件
#include <sys/stat.h>  // 包含系统状态的头文件
#ifdef HAVE_UNISTD_H
#include <unistd.h>  // 包含系统调用的头文件
#endif

/*******************
 * 导入例程 *
 *******************/

struct hwloc__nolibxml_backend_data_s {
  size_t buflen; /* 缓冲区的大小，由 backend_init() 设置 */
  char *buffer; /* 在 backend_init() 中分配和填充 */
};

typedef struct hwloc__nolibxml_import_state_data_s {
  char *tagbuffer; /* 包含下一个标签的缓冲区 */
  char *attrbuffer; /* 包含当前节点的下一个属性的缓冲区 */
  const char *tagname; /* 当前节点的标签名 */
  int closed; /* 如果当前节点是自动关闭的，则设置为1 */
} __hwloc_attribute_may_alias * hwloc__nolibxml_import_state_data_t;

static char *
hwloc__nolibxml_import_ignore_spaces(char *buffer)
{
  return buffer + strspn(buffer, " \t\n");  // 忽略缓冲区中的空格、制表符和换行符
}

static int
hwloc__nolibxml_import_next_attr(hwloc__xml_import_state_t state, char **namep, char **valuep)
{
  hwloc__nolibxml_import_state_data_t nstate = (void*) state->data;
  size_t namelen;
  size_t len, escaped;
  char *buffer, *value, *end;

  if (!nstate->attrbuffer)
    return -1;

  /* 找到属性的开始 */
  buffer = hwloc__nolibxml_import_ignore_spaces(nstate->attrbuffer);
  namelen = strspn(buffer, "abcdefghijklmnopqrstuvwxyz_");
  if (buffer[namelen] != '=' || buffer[namelen+1] != '\"')  // 如果不是属性的开始，则返回-1
    # 返回错误代码 -1
    return -1;
  # 将缓冲区的第 namelen 个字符设为字符串结束符
  buffer[namelen] = '\0';
  # 将指向名称的指针指向缓冲区
  *namep = buffer;

  # 找到值的起始位置，并对其进行转义处理
  *valuep = value = buffer+namelen+2;
  len = 0; escaped = 0;
  while (value[len+escaped] != '\"') {
    # 如果遇到转义字符 &
    if (value[len+escaped] == '&') {
      # 根据不同的转义序列进行处理
      if (!strncmp(&value[1+len+escaped], "#10;", 4)) {
    escaped += 4;
    value[len] = '\n';
      } else if (!strncmp(&value[1+len+escaped], "#13;", 4)) {
    escaped += 4;
    value[len] = '\r';
      } else if (!strncmp(&value[1+len+escaped], "#9;", 3)) {
    escaped += 3;
    value[len] = '\t';
      } else if (!strncmp(&value[1+len+escaped], "quot;", 5)) {
    escaped += 5;
    value[len] = '\"';
      } else if (!strncmp(&value[1+len+escaped], "lt;", 3)) {
    escaped += 3;
    value[len] = '<';
      } else if (!strncmp(&value[1+len+escaped], "gt;", 3)) {
    escaped += 3;
    value[len] = '>';
      } else if (!strncmp(&value[1+len+escaped], "amp;", 4)) {
    escaped += 4;
    value[len] = '&';
      } else {
    # 返回错误代码 -1
    return -1;
      }
    } else {
      value[len] = value[len+escaped];
    }
    len++;
    # 如果值的结束符未找到，返回错误代码 -1
    if (value[len+escaped] == '\0')
      return -1;
  }
  # 将值的结束符设为字符串结束符
  value[len] = '\0';

  # 找到下一个属性
  end = &value[len+escaped+1]; # 跳过结束的 "
  # 将状态的属性缓冲区指向去除空格后的 end
  nstate->attrbuffer = hwloc__nolibxml_import_ignore_spaces(end);
  # 返回成功代码 0
  return 0;
}
# 定义一个静态函数，用于在 XML 导入状态中查找子节点
static int
hwloc__nolibxml_import_find_child(hwloc__xml_import_state_t state,
                  hwloc__xml_import_state_t childstate,
                  char **tagp)
{
  # 将状态数据转换为 hwloc__nolibxml_import_state_data_t 类型
  hwloc__nolibxml_import_state_data_t nstate = (void*) state->data;
  # 将子状态数据转换为 hwloc__nolibxml_import_state_data_t 类型
  hwloc__nolibxml_import_state_data_t nchildstate = (void*) childstate->data;
  # 为标签创建缓冲区
  char *buffer = nstate->tagbuffer;
  char *end;
  char *tag;
  size_t namelen;

  # 将子状态的父状态设置为当前状态
  childstate->parent = state;
  # 将子状态的全局状态设置为当前状态的全局状态
  childstate->global = state->global;

  # 如果当前状态为自动关闭状态，则返回 0
  if (nstate->closed)
    return 0;

  # 查找标签的起始位置
  buffer = hwloc__nolibxml_import_ignore_spaces(buffer);
  if (buffer[0] != '<')
    return -1;
  buffer++;

  # 如果是闭合标签，则返回 0 并不进行进一步处理
  if (buffer[0] == '/')
    return 0;

  # 普通标签
  nchildstate->tagname = tag = buffer;

  # 查找标签的结束位置，并标记它
  end = strchr(buffer, '>');
  if (!end)
    return -1;
  end[0] = '\0';
  nchildstate->tagbuffer = end+1;

  # 处理自动关闭的标签
  if (end[-1] == '/') {
    nchildstate->closed = 1;
    end[-1] = '\0';
  } else
    nchildstate->closed = 0;

  # 查找属性
  namelen = strspn(buffer, "abcdefghijklmnopqrstuvwxyz1234567890_");

  if (buffer[namelen] == '\0') {
    # 没有属性
    nchildstate->attrbuffer = NULL;
    *tagp = tag;
    return 1;
  }

  if (buffer[namelen] != ' ')
    return -1;

  # 找到空格，可能是属性的开始
  buffer[namelen] = '\0';
  nchildstate->attrbuffer = buffer+namelen+1;
  *tagp = tag;
  return 1;
}

# 定义一个静态函数，用于关闭标签
static int
hwloc__nolibxml_import_close_tag(hwloc__xml_import_state_t state)
{
  # 将状态数据转换为 hwloc__nolibxml_import_state_data_t 类型
  hwloc__nolibxml_import_state_data_t nstate = (void*) state->data;
  # 为标签创建缓冲区
  char *buffer = nstate->tagbuffer;
  char *end;

  # 自动关闭的标签不需要处理
  if (nstate->closed)
    return 0;

  # 查找标签的起始位置
  buffer = hwloc__nolibxml_import_ignore_spaces(buffer);
  if (buffer[0] != '<')
    # 返回-1，表示未找到
    return -1;
  # 指针向后移动一位
  buffer++;

  # 寻找结束标记，并标记它，然后返回给父级
  end = strchr(buffer, '>');
  if (!end)
    # 如果未找到结束标记，返回-1
    return -1;
  end[0] = '\0';
  nstate->tagbuffer = end+1;

  # 如果是闭合标签，返回空
  if (buffer[0] != '/' || strcmp(buffer+1, nstate->tagname) )
    # 如果不是闭合标签，或者闭合标签与当前标签名不匹配，返回-1
    return -1;
  # 返回0，表示成功
  return 0;
}
# 关闭当前子节点，将子节点的标签缓冲区内容赋值给父节点的标签缓冲区
static void
hwloc__nolibxml_import_close_child(hwloc__xml_import_state_t state)
{
  # 将当前状态的数据转换为无libxml导入状态数据
  hwloc__nolibxml_import_state_data_t nstate = (void*) state->data;
  # 将父节点状态的数据转换为无libxml导入状态数据
  hwloc__nolibxml_import_state_data_t nparent = (void*) state->parent->data;
  # 将当前状态的标签缓冲区内容赋值给父节点状态的标签缓冲区
  nparent->tagbuffer = nstate->tagbuffer;
}

# 获取内容
static int
hwloc__nolibxml_import_get_content(hwloc__xml_import_state_t state,
                   const char **beginp, size_t expected_length)
{
  # 将当前状态的数据转换为无libxml导入状态数据
  hwloc__nolibxml_import_state_data_t nstate = (void*) state->data;
  # 获取当前状态的标签缓冲区
  char *buffer = nstate->tagbuffer;
  size_t length;
  char *end;

  # 如果标签已经关闭，则没有内容
  if (nstate->closed) {
    # 如果期望长度不为0，则返回-1
    if (expected_length)
      return -1;
    # 将起始指针指向空字符串
    *beginp = "";
    return 0;
  }

  # 查找下一个标签，确定内容的结束位置
  end = strchr(buffer, '<');
  if (!end)
    return -1;

  length = (size_t) (end-buffer);
  # 如果长度不等于期望长度，则返回-1
  if (length != expected_length)
    return -1;
  # 更新标签缓冲区的位置
  nstate->tagbuffer = end;
  # 将结束位置标记为0，暂时终止字符串
  *end = '\0'; 
  # 将起始指针指向内容的起始位置
  *beginp = buffer;
  return 1;
}

# 关闭内容
static void
hwloc__nolibxml_import_close_content(hwloc__xml_import_state_t state)
{
  # 将当前状态的数据转换为无libxml导入状态数据
  hwloc__nolibxml_import_state_data_t nstate = (void*) state->data;
  # 如果标签没有关闭，则将标签缓冲区的内容还原
  if (!nstate->closed)
    *nstate->tagbuffer = '<';
}

# 初始化无libxml查看器
static int
hwloc_nolibxml_look_init(struct hwloc_xml_backend_data_s *bdata,
             struct hwloc__xml_import_state_s *state)
{
  # 将当前状态的数据转换为无libxml导入状态数据
  hwloc__nolibxml_import_state_data_t nstate = (void*) state->data;
  # 将无libxml后端数据转换为无libxml后端数据
  struct hwloc__nolibxml_backend_data_s *nbdata = bdata->data;
  unsigned major, minor;
  char *end;
  char *buffer = nbdata->buffer;
  const char *tagname;

  # 断言状态数据的大小不超过状态数据的大小
  HWLOC_BUILD_ASSERT(sizeof(*nstate) <= sizeof(state->data));

  # 跳过头部
  while (!strncmp(buffer, "<?xml ", 6) || !strncmp(buffer, "<!DOCTYPE ", 10)) {
    buffer = strchr(buffer, '\n');
    if (!buffer)
      goto failed;
    buffer++;
  }

  # 查找拓扑标签
  if (sscanf(buffer, "<topology version=\"%u.%u\">", &major, &minor) == 2) {
    # 设置bdata结构体中的主版本号为major
    bdata->version_major = major;
    # 设置bdata结构体中的次版本号为minor
    bdata->version_minor = minor;
    # 在buffer中查找'>'字符，并返回其位置的下一个位置，赋值给end
    end = strchr(buffer, '>') + 1;
    # 设置tagname为"topology"
    tagname = "topology";
  } else if (!strncmp(buffer, "<topology>", 10)) {
    # 如果buffer中的前10个字符与"<topology>"相同，则执行以下操作
    # 设置bdata结构体中的主版本号为1
    bdata->version_major = 1;
    # 设置bdata结构体中的次版本号为0
    bdata->version_minor = 0;
    # 设置end为buffer加上10
    end = buffer + 10;
    # 设置tagname为"topology"
    tagname = "topology";
  } else if (!strncmp(buffer, "<root>", 6)) {
    # 如果buffer中的前6个字符与"<root>"相同，则执行以下操作
    # 设置bdata结构体中的主版本号为0
    bdata->version_major = 0;
    # 设置bdata结构体中的次版本号为9
    bdata->version_minor = 9;
    # 设置end为buffer加上6
    end = buffer + 6;
    # 设置tagname为"root"
    tagname = "root";
  } else
    # 如果以上条件都不满足，则跳转到failed标签处
    goto failed;

  # 设置state->global->next_attr为hwloc__nolibxml_import_next_attr
  state->global->next_attr = hwloc__nolibxml_import_next_attr;
  # 设置state->global->find_child为hwloc__nolibxml_import_find_child
  state->global->find_child = hwloc__nolibxml_import_find_child;
  # 设置state->global->close_tag为hwloc__nolibxml_import_close_tag
  state->global->close_tag = hwloc__nolibxml_import_close_tag;
  # 设置state->global->close_child为hwloc__nolibxml_import_close_child
  state->global->close_child = hwloc__nolibxml_import_close_child;
  # 设置state->global->get_content为hwloc__nolibxml_import_get_content
  state->global->get_content = hwloc__nolibxml_import_get_content;
  # 设置state->global->close_content为hwloc__nolibxml_import_close_content
  state->global->close_content = hwloc__nolibxml_import_close_content;
  # 设置state->parent为NULL
  state->parent = NULL;
  # 设置nstate->closed为0
  nstate->closed = 0;
  # 设置nstate->tagbuffer为end
  nstate->tagbuffer = end;
  # 设置nstate->tagname为tagname
  nstate->tagname = tagname;
  # 返回0，表示成功
  return 0; /* success */

 failed:
  # 返回-1，表示失败
  return -1; /* failed */
/* 用于在导入结束时（提前清理事务）或者在 load 失败的其他原因下由 backend_exit() 调用。 */
static void
hwloc_nolibxml_free_buffers(struct hwloc_xml_backend_data_s *bdata)
{
  struct hwloc__nolibxml_backend_data_s *nbdata = bdata->data;
  // 如果缓冲区不为空，则释放缓冲区内存并将指针置为空
  if (nbdata->buffer) {
    free(nbdata->buffer);
    nbdata->buffer = NULL;
  }
}

static void
hwloc_nolibxml_look_done(struct hwloc_xml_backend_data_s *bdata, int result)
{
  // 调用 hwloc_nolibxml_free_buffers() 释放缓冲区
  hwloc_nolibxml_free_buffers(bdata);

  // 如果解析结果小于 0 并且启用了详细输出，则打印错误信息
  if (result < 0 && hwloc__xml_verbose())
    fprintf(stderr, "Failed to parse XML input with the minimalistic parser. If it was not\n"
        "generated by hwloc, try enabling full XML support with libxml2.\n");
}

/* 后端例程 */
static void
hwloc_nolibxml_backend_exit(struct hwloc_xml_backend_data_s *bdata)
{
  struct hwloc__nolibxml_backend_data_s *nbdata = bdata->data;
  // 调用 hwloc_nolibxml_free_buffers() 释放缓冲区
  hwloc_nolibxml_free_buffers(bdata);
  // 释放 nbdata 指针指向的内存
  free(nbdata);
}

static int
hwloc_nolibxml_read_file(const char *xmlpath, char **bufferp, size_t *buflenp)
{
  FILE * file;
  size_t buflen, offset, readlen;
  struct stat statbuf;
  char *buffer, *tmp;
  size_t ret;

  // 如果 xmlpath 是 "-"，则将其替换为 "/dev/stdin"
  if (!strcmp(xmlpath, "-"))
    xmlpath = "/dev/stdin";

  // 以只读方式打开文件
  file = fopen(xmlpath, "r");
  if (!file)
    goto out;

  /* 对于常规文件，找到所需的缓冲区大小，或者在未知情况下使用 4k，如果需要的话，我们稍后会重新分配 */
  buflen = 4096;
  if (!stat(xmlpath, &statbuf))
    if (S_ISREG(statbuf.st_mode))
      buflen = statbuf.st_size+1; /* 一个额外的字节，以便第一个 fread() 也能读到 EOF */

  // 分配缓冲区内存，多分配一个字节用于结束符 \0
  buffer = malloc(buflen+1);
  if (!buffer)
    goto out_with_file;

  offset = 0; readlen = buflen;
  while (1) {
    // 从文件中读取数据到缓冲区
    ret = fread(buffer+offset, 1, readlen, file);

    offset += ret;
    buffer[offset] = 0;

    // 如果读取的字节数不等于指定的长度，则跳出循环
    if (ret != readlen)
      break;

    // 将缓冲区大小翻倍，并重新分配内存
    buflen *= 2;
    tmp = realloc(buffer, buflen+1);
    if (!tmp)
      goto out_with_buffer;
    buffer = tmp;
    # 将 buflen 的值除以 2，得到读取长度 readlen
    readlen = buflen/2;
  }

  # 关闭文件
  fclose(file);
  # 将指向 buffer 的指针赋给 bufferp
  *bufferp = buffer;
  # 将偏移量加 1 赋给 buflenp
  *buflenp = offset+1;
  # 返回 0，表示成功
  return 0;

 out_with_buffer:
  # 释放内存，释放 buffer 指向的内存空间
  free(buffer);
 out_with_file:
  # 关闭文件
  fclose(file);
 out:
  # 返回 -1，表示失败
  return -1;
}
# 初始化不使用libxml的后端数据结构
static int
hwloc_nolibxml_backend_init(struct hwloc_xml_backend_data_s *bdata,
                const char *xmlpath, const char *xmlbuffer, int xmlbuflen)
{
  # 为不使用libxml的后端数据结构分配内存空间
  struct hwloc__nolibxml_backend_data_s *nbdata = malloc(sizeof(*nbdata));

  # 如果内存分配失败，跳转到out标签
  if (!nbdata)
    goto out;
  # 将分配的数据结构赋值给bdata的data字段
  bdata->data = nbdata;

  # 如果xmlbuffer不为空
  if (xmlbuffer) {
    # 为nbdata的buffer字段分配内存空间
    nbdata->buffer = malloc(xmlbuflen+1);
    # 如果内存分配失败，跳转到out_with_nbdata标签
    if (!nbdata->buffer)
      goto out_with_nbdata;
    # 将xmlbuffer的内容复制到nbdata的buffer字段中
    memcpy(nbdata->buffer, xmlbuffer, xmlbuflen);
    # 在buffer末尾添加'\0'，表示字符串结束
    nbdata->buffer[xmlbuflen] = '\0';

  } else {
    # 调用hwloc_nolibxml_read_file函数读取xmlpath指定的文件内容到nbdata的buffer字段中
    int err = hwloc_nolibxml_read_file(xmlpath, &nbdata->buffer, &nbdata->buflen);
    # 如果读取失败，跳转到out_with_nbdata标签
    if (err < 0)
      goto out_with_nbdata;
  }

  # 设置bdata的look_init、look_done和backend_exit字段为相应的函数
  bdata->look_init = hwloc_nolibxml_look_init;
  bdata->look_done = hwloc_nolibxml_look_done;
  bdata->backend_exit = hwloc_nolibxml_backend_exit;
  # 返回0表示初始化成功
  return 0;

out_with_nbdata:
  # 释放nbdata的内存空间
  free(nbdata);
out:
  # 返回-1表示初始化失败
  return -1;
}

# 导入差异
static int
hwloc_nolibxml_import_diff(struct hwloc__xml_import_state_s *state,
               const char *xmlpath, const char *xmlbuffer, int xmlbuflen,
               hwloc_topology_diff_t *firstdiffp, char **refnamep)
{
  # 获取state的数据结构
  hwloc__nolibxml_import_state_data_t nstate = (void*) state->data;
  struct hwloc__xml_import_state_s childstate;
  char *refname = NULL;
  char *buffer, *tmp, *tag;
  size_t buflen;
  int ret;

  # 断言nstate的大小不超过state->data的大小
  HWLOC_BUILD_ASSERT(sizeof(*nstate) <= sizeof(state->data));

  # 如果xmlbuffer不为空
  if (xmlbuffer) {
    # 为buffer分配内存空间
    buffer = malloc(xmlbuflen);
    # 如果内存分配失败，跳转到out标签
    if (!buffer)
      goto out;
    # 将xmlbuffer的内容复制到buffer中
    memcpy(buffer, xmlbuffer, xmlbuflen);
    # 设置buflen为xmlbuflen
    buflen = xmlbuflen;

  } else {
    # 调用hwloc_nolibxml_read_file函数读取xmlpath指定的文件内容到buffer中
    ret = hwloc_nolibxml_read_file(xmlpath, &buffer, &buflen);
    # 如果读取失败，跳转到out标签
    if (ret < 0)
      goto out;
  }

  # 跳过头部信息
  tmp = buffer;
  while (!strncmp(tmp, "<?xml ", 6) || !strncmp(tmp, "<!DOCTYPE ", 10)) {
    # 查找换行符的位置
    tmp = strchr(tmp, '\n');
    # 如果没有找到换行符，跳转到out_with_buffer标签
    if (!tmp)
      goto out_with_buffer;
  // 递增 tmp 变量的值
  tmp++;
  // 设置 state 全局对象的下一个属性为 hwloc__nolibxml_import_next_attr 函数
  state->global->next_attr = hwloc__nolibxml_import_next_attr;
  // 设置 state 全局对象的查找子节点函数为 hwloc__nolibxml_import_find_child 函数
  state->global->find_child = hwloc__nolibxml_import_find_child;
  // 设置 state 全局对象的关闭标签函数为 hwloc__nolibxml_import_close_tag 函数
  state->global->close_tag = hwloc__nolibxml_import_close_tag;
  // 设置 state 全局对象的关闭子节点函数为 hwloc__nolibxml_import_close_child 函数
  state->global->close_child = hwloc__nolibxml_import_close_child;
  // 设置 state 全局对象的获取内容函数为 hwloc__nolibxml_import_get_content 函数
  state->global->get_content = hwloc__nolibxml_import_get_content;
  // 设置 state 全局对象的关闭内容函数为 hwloc__nolibxml_import_close_content 函数
  state->global->close_content = hwloc__nolibxml_import_close_content;
  // 设置 state 的父节点为 NULL
  state->parent = NULL;
  // 设置 nstate 的关闭状态为 0
  nstate->closed = 0;
  // 设置 nstate 的标签缓冲区为 tmp 变量的值
  nstate->tagbuffer = tmp;
  // 设置 nstate 的标签名为 NULL
  nstate->tagname = NULL;
  // 设置 nstate 的属性缓冲区为 NULL
  nstate->attrbuffer = NULL;

  /* 查找根节点 */
  // 调用 hwloc__nolibxml_import_find_child 函数查找子节点
  ret = hwloc__nolibxml_import_find_child(state, &childstate, &tag);
  // 如果查找失败，则跳转到 out_with_buffer 标签
  if (ret < 0)
    goto out_with_buffer;
  // 如果找不到标签或者标签不是 "topologydiff"，则跳转到 out_with_buffer 标签
  if (!tag || strcmp(tag, "topologydiff"))
    goto out_with_buffer;

  while (1) {
    char *attrname, *attrvalue;
    // 调用 hwloc__nolibxml_import_next_attr 函数获取下一个属性的名称和值
    if (hwloc__nolibxml_import_next_attr(&childstate, &attrname, &attrvalue) < 0)
      break;
    // 如果属性名是 "refname"，则释放 refname 变量的内存并将其赋值为属性值
    if (!strcmp(attrname, "refname")) {
      free(refname);
      refname = strdup(attrvalue);
    } else
      // 否则跳转到 out_with_buffer 标签
      goto out_with_buffer;
  }

  // 调用 hwloc__xml_import_diff 函数解析子节点的内容
  ret = hwloc__xml_import_diff(&childstate, firstdiffp);
  // 如果 refnamep 不为空且 ret 为假，则将 refnamep 指向的指针赋值为 refname
  if (refnamep && !ret)
    *refnamep = refname;
  else
    // 否则释放 refname 变量的内存
    free(refname);

  // 释放 buffer 变量的内存
  free(buffer);
  // 返回解析结果
  return ret;
out_with_buffer:
  // 释放缓冲区内存
  free(buffer);
  // 释放引用名称内存
  free(refname);
out:
  // 返回错误码
  return -1;
}

/*******************
 * Export routines *
 *******************/

typedef struct hwloc__nolibxml_export_state_data_s {
  char *buffer; /* (moving) buffer where to write */ 
  size_t written; /* how many bytes were written (or would have be written if not truncated) */
  size_t remaining; /* how many bytes are still available in the buffer */
  unsigned indent; /* indentation level for the next line */
  unsigned nr_children;
  unsigned has_content;
} __hwloc_attribute_may_alias * hwloc__nolibxml_export_state_data_t;

static void
hwloc__nolibxml_export_update_buffer(hwloc__nolibxml_export_state_data_t ndata, int res)
{
  if (res >= 0) {
    // 更新已写入字节数
    ndata->written += res;
    // 如果写入字节数大于等于剩余字节数，则更新写入字节数为剩余字节数减一
    if (res >= (int) ndata->remaining)
      res = ndata->remaining>0 ? (int)ndata->remaining-1 : 0;
    // 移动缓冲区指针
    ndata->buffer += res;
    // 更新剩余字节数
    ndata->remaining -= res;
  }
}

static char *
hwloc__nolibxml_export_escape_string(const char *src)
{
  size_t fulllen, sublen;
  char *escaped, *dst;

  // 计算源字符串长度
  fulllen = strlen(src);

  // 计算不包含特殊字符的子字符串长度
  sublen = strcspn(src, "\n\r\t\"<>&");
  // 如果子字符串长度等于源字符串长度，则无需转义，直接返回空指针
  if (sublen == fulllen)
    return NULL; /* nothing to escape */

  // 分配转义后字符串的内存空间
  escaped = malloc(fulllen*6+1); /* escaped chars are replaced by at most 6 char */
  dst = escaped;

  // 复制不包含特殊字符的子字符串
  memcpy(dst, src, sublen);
  src += sublen;
  dst += sublen;

  // 循环处理源字符串中的特殊字符
  while (*src) {
    int replen;
    switch (*src) {
    case '\n': strcpy(dst, "&#10;");  replen=5; break;
    case '\r': strcpy(dst, "&#13;");  replen=5; break;
    case '\t': strcpy(dst, "&#9;");   replen=4; break;
    case '\"': strcpy(dst, "&quot;"); replen=6; break;
    case '<':  strcpy(dst, "&lt;");   replen=4; break;
    case '>':  strcpy(dst, "&gt;");   replen=4; break;
    case '&':  strcpy(dst, "&amp;");  replen=5; break;
    default: replen=0; break;
    }
    dst+=replen; src++;

    // 计算不包含特殊字符的子字符串长度
    sublen = strcspn(src, "\n\r\t\"<>&");
    // 复制不包含特殊字符的子字符串
    memcpy(dst, src, sublen);
    src += sublen;
    dst += sublen;
  }

  // 添加字符串结束符
  *dst = 0;
  return escaped;
}

static void
# 创建一个新的子节点，并更新父节点的状态
hwloc__nolibxml_export_new_child(hwloc__xml_export_state_t parentstate,
                 hwloc__xml_export_state_t state,
                 const char *name)
{
  # 将父节点的数据转换为无库XML导出状态数据
  hwloc__nolibxml_export_state_data_t npdata = (void *) parentstate->data;
  # 将当前节点的数据转换为无库XML导出状态数据
  hwloc__nolibxml_export_state_data_t ndata = (void *) state->data;
  # 初始化结果变量
  int res;

  # 断言父节点没有内容
  assert(!npdata->has_content);
  # 如果父节点没有子节点
  if (!npdata->nr_children) {
    # 将结束标签写入父节点的缓冲区
    res = hwloc_snprintf(npdata->buffer, npdata->remaining, ">\n");
    # 更新父节点的缓冲区
    hwloc__nolibxml_export_update_buffer(npdata, res);
  }
  # 父节点的子节点数量加一
  npdata->nr_children++;

  # 设置当前节点的父节点
  state->parent = parentstate;
  # 将当前节点的新子节点函数指针指向父节点的新子节点函数
  state->new_child = parentstate->new_child;
  # 将当前节点的新属性函数指针指向父节点的新属性函数
  state->new_prop = parentstate->new_prop;
  # 将当前节点的添加内容函数指针指向父节点的添加内容函数
  state->add_content = parentstate->add_content;
  # 将当前节点的结束对象函数指针指向父节点的结束对象函数
  state->end_object = parentstate->end_object;
  # 将当前节点的全局标志设置为父节点的全局标志
  state->global = parentstate->global;

  # 将当前节点的缓冲区、已写入长度、剩余长度、缩进等信息设置为父节点的对应信息
  ndata->buffer = npdata->buffer;
  ndata->written = npdata->written;
  ndata->remaining = npdata->remaining;
  ndata->indent = npdata->indent + 2;

  # 将当前节点的子节点数量和内容标志重置为0
  ndata->nr_children = 0;
  ndata->has_content = 0;

  # 将当前节点的开始标签写入缓冲区
  res = hwloc_snprintf(ndata->buffer, ndata->remaining, "%*s<%s", (int) npdata->indent, "", name);
  # 更新当前节点的缓冲区
  hwloc__nolibxml_export_update_buffer(ndata, res);
}

# 创建一个新的属性，并将其写入当前节点的缓冲区
static void
hwloc__nolibxml_export_new_prop(hwloc__xml_export_state_t state, const char *name, const char *value)
{
  # 将当前节点的数据转换为无库XML导出状态数据
  hwloc__nolibxml_export_state_data_t ndata = (void *) state->data;
  # 对属性值进行转义处理
  char *escaped = hwloc__nolibxml_export_escape_string(value);
  # 将属性名和转义后的属性值写入当前节点的缓冲区
  int res = hwloc_snprintf(ndata->buffer, ndata->remaining, " %s=\"%s\"", name, escaped ? (const char *) escaped : value);
  # 更新当前节点的缓冲区
  hwloc__nolibxml_export_update_buffer(ndata, res);
  # 释放转义后的属性值内存
  free(escaped);
}

# 结束当前节点的导出，并将结束标签写入当前节点的缓冲区
static void
hwloc__nolibxml_export_end_object(hwloc__xml_export_state_t state, const char *name)
{
  # 将当前节点的数据转换为无库XML导出状态数据
  hwloc__nolibxml_export_state_data_t ndata = (void *) state->data;
  # 将当前节点的父节点的数据转换为无库XML导出状态数据
  hwloc__nolibxml_export_state_data_t npdata = (void *) state->parent->data;
  # 初始化结果变量
  int res;

  # 断言当前节点既没有内容也没有子节点
  assert (!(ndata->has_content && ndata->nr_children));
  # 如果当前节点有内容
  if (ndata->has_content) {
  # 使用给定的数据格式化字符串，将结果存储在指定的缓冲区中
  res = hwloc_snprintf(ndata->buffer, ndata->remaining, "</%s>\n", name);
  # 如果子节点数量大于0
  } else if (ndata->nr_children) {
    # 使用给定的数据格式化字符串，将结果存储在指定的缓冲区中，包括缩进
    res = hwloc_snprintf(ndata->buffer, ndata->remaining, "%*s</%s>\n", (int) npdata->indent, "", name);
  # 如果子节点数量为0
  } else {
    # 使用给定的数据格式化字符串，将结果存储在指定的缓冲区中，表示空标签
    res = hwloc_snprintf(ndata->buffer, ndata->remaining, "/>\n");
  }
  # 更新导出数据的缓冲区
  hwloc__nolibxml_export_update_buffer(ndata, res);

  # 更新父节点的缓冲区、已写入数据和剩余空间
  npdata->buffer = ndata->buffer;
  npdata->written = ndata->written;
  npdata->remaining = ndata->remaining;
}

static void
hwloc__nolibxml_export_add_content(hwloc__xml_export_state_t state, const char *buffer, size_t length __hwloc_attribute_unused)
{
  // 将状态数据转换为特定类型的指针
  hwloc__nolibxml_export_state_data_t ndata = (void *) state->data;
  int res;

  // 断言确保没有子节点
  assert(!ndata->nr_children);
  // 如果没有内容，则添加结束标记
  if (!ndata->has_content) {
    res = hwloc_snprintf(ndata->buffer, ndata->remaining, ">");
    hwloc__nolibxml_export_update_buffer(ndata, res);
  }
  // 标记已有内容
  ndata->has_content = 1;

  // 将内容添加到缓冲区
  res = hwloc_snprintf(ndata->buffer, ndata->remaining, "%s", buffer);
  hwloc__nolibxml_export_update_buffer(ndata, res);
}

static size_t
hwloc___nolibxml_prepare_export(hwloc_topology_t topology, struct hwloc__xml_export_data_s *edata,
                char *xmlbuffer, int buflen, unsigned long flags)
{
  struct hwloc__xml_export_state_s state, childstate;
  // 将状态数据转换为特定类型的指针
  hwloc__nolibxml_export_state_data_t ndata = (void *) &state.data;
  int v1export = flags & HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1;
  int res;

  // 编译时断言确保状态数据大小不超过特定大小
  HWLOC_BUILD_ASSERT(sizeof(*ndata) <= sizeof(state.data));

  // 设置状态对象的各个回调函数
  state.new_child = hwloc__nolibxml_export_new_child;
  state.new_prop = hwloc__nolibxml_export_new_prop;
  state.add_content = hwloc__nolibxml_export_add_content;
  state.end_object = hwloc__nolibxml_export_end_object;
  state.global = edata;

  // 初始化状态数据
  ndata->indent = 0;
  ndata->written = 0;
  ndata->buffer = xmlbuffer;
  ndata->remaining = buflen;

  ndata->nr_children = 1; // 不关闭前一个标签，因为拓扑标签还未打开
  ndata->has_content = 0;

  // 将 XML 头部信息添加到缓冲区
  res = hwloc_snprintf(ndata->buffer, ndata->remaining,
         "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n"
         "<!DOCTYPE topology SYSTEM \"%s\">\n", v1export ? "hwloc.dtd" : "hwloc2.dtd");
  hwloc__nolibxml_export_update_buffer(ndata, res);
  // 创建拓扑标签
  hwloc__nolibxml_export_new_child(&state, &childstate, "topology");
  // 如果不是导出为 V1 版本的 XML，则...
    # 使用指定属性创建新的属性节点，属性名为"version"，属性值为"2.0"
    hwloc__nolibxml_export_new_prop(&childstate, "version", "2.0");
    # 导出拓扑结构到 XML 格式，并添加到子状态中
    hwloc__xml_export_topology (&childstate, topology, flags);
    # 结束拓扑结构的导出
    hwloc__nolibxml_export_end_object(&childstate, "topology");
    # 返回已写入数据的长度加1，表示字符串结束符'\0'
    return ndata->written+1; /* ending \0 */
# 定义一个静态函数，将拓扑结构导出到内存缓冲区中，使用nolibxml库
static int
hwloc_nolibxml_export_buffer(hwloc_topology_t topology, struct hwloc__xml_export_data_s *edata,
                 char **bufferp, int *buflenp, unsigned long flags)
{
  char *buffer;  # 定义一个字符指针变量buffer
  size_t bufferlen, res;  # 定义两个size_t类型的变量bufferlen和res

  bufferlen = 16384;  # 初始化bufferlen为16384，作为默认缓冲区大小
  buffer = malloc(bufferlen);  # 分配bufferlen大小的内存空间给buffer
  if (!buffer)  # 如果内存分配失败
    return -1;  # 返回-1表示失败
  res = hwloc___nolibxml_prepare_export(topology, edata, buffer, (int)bufferlen, flags);  # 调用函数准备导出拓扑结构到buffer中

  if (res > bufferlen) {  # 如果实际导出的数据大小超过了bufferlen
    char *tmp = realloc(buffer, res);  # 重新分配res大小的内存空间给buffer
    if (!tmp) {  # 如果内存分配失败
      free(buffer);  # 释放之前分配的内存空间
      return -1;  # 返回-1表示失败
    }
    buffer = tmp;  # 将重新分配的内存空间赋值给buffer
    hwloc___nolibxml_prepare_export(topology, edata, buffer, (int)res, flags);  # 再次调用函数准备导出拓扑结构到buffer中
  }

  *bufferp = buffer;  # 将buffer的地址赋值给bufferp指针
  *buflenp = (int)res;  # 将res的值赋值给buflenp指针
  return 0;  # 返回0表示成功
}

# 定义一个静态函数，将拓扑结构导出到文件中，使用nolibxml库
static int
hwloc_nolibxml_export_file(hwloc_topology_t topology, struct hwloc__xml_export_data_s *edata,
               const char *filename, unsigned long flags)
{
  FILE *file;  # 定义一个文件指针变量file
  char *buffer;  # 定义一个字符指针变量buffer
  int bufferlen;  # 定义一个整型变量bufferlen
  int ret;  # 定义一个整型变量ret

  ret = hwloc_nolibxml_export_buffer(topology, edata, &buffer, &bufferlen, flags);  # 调用函数将拓扑结构导出到内存缓冲区中
  if (ret < 0)  # 如果导出失败
    return -1;  # 返回-1表示失败

  if (!strcmp(filename, "-")) {  # 如果文件名为"-"
    file = stdout;  # 将标准输出赋值给file
  } else {
    file = fopen(filename, "w");  # 以写入模式打开文件
    if (!file) {  # 如果文件打开失败
      free(buffer);  # 释放内存空间
      return -1;  # 返回-1表示失败
    }
  }

  ret = (int)fwrite(buffer, 1, bufferlen-1 /* don't write the ending \0 */, file);  # 将buffer中的数据写入文件
  if (ret == bufferlen-1) {  # 如果写入成功
    ret = 0;  # 将ret赋值为0表示成功
  } else {
    errno = ferror(file);  # 获取文件错误码
    ret = -1;  # 返回-1表示失败
  }

  free(buffer);  # 释放内存空间

  if (file != stdout)  # 如果文件不是标准输出
    fclose(file);  # 关闭文件
  return ret;  # 返回ret表示结果
}

# 定义一个静态函数，准备导出拓扑结构的差异到xmlbuffer中
static size_t
hwloc___nolibxml_prepare_export_diff(hwloc_topology_diff_t diff, const char *refname, char *xmlbuffer, int buflen)
{
  // 定义两个 XML 导出状态结构体
  struct hwloc__xml_export_state_s state, childstate;
  // 将 state.data 强制转换为 hwloc__nolibxml_export_state_data_t 类型，并赋值给 ndata
  hwloc__nolibxml_export_state_data_t ndata = (void *) &state.data;
  // 定义一个整型变量 res
  int res;

  // 断言 ndata 的大小不超过 state.data 的大小
  HWLOC_BUILD_ASSERT(sizeof(*ndata) <= sizeof(state.data));

  // 初始化 state 结构体的各个成员函数
  state.new_child = hwloc__nolibxml_export_new_child;
  state.new_prop = hwloc__nolibxml_export_new_prop;
  state.add_content = hwloc__nolibxml_export_add_content;
  state.end_object = hwloc__nolibxml_export_end_object;
  state.global = NULL;

  // 初始化 ndata 结构体的各个成员变量
  ndata->indent = 0;
  ndata->written = 0;
  ndata->buffer = xmlbuffer;
  ndata->remaining = buflen;

  // 设置 ndata 结构体的 nr_children 和 has_content 成员变量
  ndata->nr_children = 1; // 不关闭前一个标签，因为拓扑标签还未打开
  ndata->has_content = 0;

  // 使用 hwloc_snprintf 将字符串写入 ndata->buffer 中，并更新 ndata->written
  res = hwloc_snprintf(ndata->buffer, ndata->remaining,
         "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n"
         "<!DOCTYPE topologydiff SYSTEM \"hwloc2-diff.dtd\">\n");
  hwloc__nolibxml_export_update_buffer(ndata, res);
  // 调用 hwloc__nolibxml_export_new_child 函数打开 topologydiff 标签
  hwloc__nolibxml_export_new_child(&state, &childstate, "topologydiff");
  // 如果 refname 存在，则将其作为属性添加到 childstate 中
  if (refname)
    hwloc__nolibxml_export_new_prop(&childstate, "refname", refname);
  // 调用 hwloc__xml_export_diff 函数导出 diff
  hwloc__xml_export_diff (&childstate, diff);
  // 调用 hwloc__nolibxml_export_end_object 函数关闭 topologydiff 标签
  hwloc__nolibxml_export_end_object(&childstate, "topologydiff");

  // 返回 ndata->written+1
  return ndata->written+1;
}

// 定义一个函数，将拓扑差异导出到缓冲区
static int
hwloc_nolibxml_export_diff_buffer(hwloc_topology_diff_t diff, const char *refname, char **bufferp, int *buflenp)
{
  // 定义缓冲区和缓冲区长度
  char *buffer;
  size_t bufferlen, res;

  // 初始化缓冲区长度为 16384
  bufferlen = 16384; // 随机猜测一个足够大的默认值
  // 分配缓冲区内存
  buffer = malloc(bufferlen);
  // 如果内存分配失败，则返回 -1
  if (!buffer)
    return -1;
  // 调用 hwloc___nolibxml_prepare_export_diff 函数准备导出 diff，并将结果保存到 buffer 中
  res = hwloc___nolibxml_prepare_export_diff(diff, refname, buffer, (int)bufferlen);

  // 如果结果超过了缓冲区长度，则重新分配更大的内存
  if (res > bufferlen) {
    char *tmp = realloc(buffer, res);
    if (!tmp) {
      free(buffer);
      return -1;
    }
    buffer = tmp;
    hwloc___nolibxml_prepare_export_diff(diff, refname, buffer, (int)res);
  }

  // 将结果保存到 bufferp 和 buflenp 中，并返回 0
  *bufferp = buffer;
  *buflenp = (int)res;
  return 0;
}

// 定义一个函数，将拓扑差异导出到文件
static int
hwloc_nolibxml_export_diff_file(hwloc_topology_diff_t diff, const char *refname, const char *filename)
{
  // 声明文件指针
  FILE *file;
  // 声明字符指针buffer和整型变量bufferlen
  char *buffer;
  int bufferlen;
  int ret;

  // 调用hwloc_nolibxml_export_diff_buffer函数，将结果存储在buffer中，bufferlen中存储buffer的长度
  ret = hwloc_nolibxml_export_diff_buffer(diff, refname, &buffer, &bufferlen);
  // 如果ret小于0，返回-1
  if (ret < 0)
    return -1;

  // 如果filename等于"-"，将file指向标准输出流stdout
  if (!strcmp(filename, "-")) {
    file = stdout;
  } else {
    // 否则，以写入模式打开文件，并将文件指针存储在file中
    file = fopen(filename, "w");
    // 如果文件指针为NULL，释放buffer并返回-1
    if (!file) {
      free(buffer);
      return -1;
    }
  }

  // 将buffer中的内容写入file中，长度为bufferlen-1（不包括结尾的\0）
  ret = (int)fwrite(buffer, 1, bufferlen-1 /* don't write the ending \0 */, file);
  // 如果成功写入的字节数等于bufferlen-1，将ret设为0
  if (ret == bufferlen-1) {
    ret = 0;
  } else {
    // 否则，将errno设为file的错误标志，ret设为-1
    errno = ferror(file);
    ret = -1;
  }

  // 释放buffer的内存
  free(buffer);

  // 如果file不等于stdout，关闭文件
  if (file != stdout)
    fclose(file);
  // 返回ret
  return ret;
}

// 释放xmlbuffer的内存
static void
hwloc_nolibxml_free_buffer(void *xmlbuffer)
{
  free(xmlbuffer);
}

/*************
 * Callbacks *
 *************/

// 定义hwloc_xml_nolibxml_callbacks结构体，包含各种回调函数
static struct hwloc_xml_callbacks hwloc_xml_nolibxml_callbacks = {
  hwloc_nolibxml_backend_init,
  hwloc_nolibxml_export_file,
  hwloc_nolibxml_export_buffer,
  hwloc_nolibxml_free_buffer,
  hwloc_nolibxml_import_diff,
  hwloc_nolibxml_export_diff_file,
  hwloc_nolibxml_export_diff_buffer
};

// 定义hwloc_nolibxml_xml_component结构体，包含指向hwloc_xml_nolibxml_callbacks的指针
static struct hwloc_xml_component hwloc_nolibxml_xml_component = {
  &hwloc_xml_nolibxml_callbacks,
  NULL
};

// 定义hwloc_xml_nolibxml_component结构体，包含指向hwloc_nolibxml_xml_component的指针
const struct hwloc_component hwloc_xml_nolibxml_component = {
  HWLOC_COMPONENT_ABI,
  NULL, NULL,
  HWLOC_COMPONENT_TYPE_XML,
  0,
  &hwloc_nolibxml_xml_component
};
```