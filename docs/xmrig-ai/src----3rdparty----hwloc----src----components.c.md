# `xmrig\src\3rdparty\hwloc\src\components.c`

```
/*
 * 版权声明，版权归 Inria 所有
 * 版权声明，版权归 2012 年 Université Bordeaux 所有
 * 请参阅顶层目录中的 COPYING 文件
 */

#include "private/autogen/config.h" // 包含自动生成的配置文件
#include "hwloc.h" // 包含 hwloc 头文件
#include "private/private.h" // 包含私有头文件
#include "private/xml.h" // 包含 XML 处理相关的私有头文件
#include "private/misc.h" // 包含私有杂项头文件

#define HWLOC_COMPONENT_STOP_NAME "stop" // 定义组件停止的名称
#define HWLOC_COMPONENT_EXCLUDE_CHAR '-' // 定义组件排除字符
#define HWLOC_COMPONENT_SEPS "," // 定义组件分隔符
#define HWLOC_COMPONENT_PHASESEP_CHAR ':' // 定义组件阶段分隔符

/* 所有已注册的发现组件的列表，按优先级排序，优先级高的在前。
 * noos 是最后一个，因为它的优先级是 0。
 * 其他的优先级是 10。
 */
static struct hwloc_disc_component * hwloc_disc_components = NULL;

static unsigned hwloc_components_users = 0; // 使用组件的用户数量，第一个初始化，最后一个销毁

static int hwloc_components_verbose = 0; // 组件的详细信息输出开关
#ifdef HWLOC_HAVE_PLUGINS
static int hwloc_plugins_verbose = 0; // 插件的详细信息输出开关
static const char * hwloc_plugins_blacklist = NULL; // 插件的黑名单
#endif

/* hwloc_components_mutex 用于串行化：
 * - 加载/卸载插件，以及修改 hwloc_plugins 列表
 * - 调用 ltdl，包括在 hwloc_check_plugin_namespace() 中的调用
 * - 使用 hwloc_disc_component_register() 和 hwloc_xml_callbacks_register() 注册组件
 */
#ifdef HWLOC_WIN_SYS
/* 在 Windows 上基本的互斥锁，使用 InterlockedCompareExchange() 实现，
 * 远非完美，但易于维护，而且在实际情况下永远不会需要这段代码。
 */
#include <windows.h>
static LONG hwloc_components_mutex = 0; // Windows 上的互斥锁
#define HWLOC_COMPONENTS_LOCK() do {                        \
  while (InterlockedCompareExchange(&hwloc_components_mutex, 1, 0) != 0)    \
    SwitchToThread();                                \
} while (0)
#define HWLOC_COMPONENTS_UNLOCK() do {                        \
  assert(hwloc_components_mutex == 1);                        \
  hwloc_components_mutex = 0;                            \
} while (0)

#elif defined HWLOC_HAVE_PTHREAD_MUTEX
/* 如果可用，使用 pthread 互斥锁（除了在 Windows 上） */
#include <pthread.h>
static pthread_mutex_t hwloc_components_mutex = PTHREAD_MUTEX_INITIALIZER;
#define HWLOC_COMPONENTS_LOCK() pthread_mutex_lock(&hwloc_components_mutex)
#define HWLOC_COMPONENTS_UNLOCK() pthread_mutex_unlock(&hwloc_components_mutex)

#else /* HWLOC_WIN_SYS || HWLOC_HAVE_PTHREAD_MUTEX */
#error No mutex implementation available
#endif


#ifdef HWLOC_HAVE_PLUGINS

#ifdef HWLOC_HAVE_LTDL
/* ltdl-based plugin load */
#include <ltdl.h>
typedef lt_dlhandle hwloc_dlhandle;
#define hwloc_dlinit lt_dlinit
#define hwloc_dlexit lt_dlexit
#define hwloc_dlopenext lt_dlopenext
#define hwloc_dlclose lt_dlclose
#define hwloc_dlerror lt_dlerror
#define hwloc_dlsym lt_dlsym
#define hwloc_dlforeachfile lt_dlforeachfile

#else /* !HWLOC_HAVE_LTDL */
/* no-ltdl plugin load relies on less portable libdl */
#include <dlfcn.h>
typedef void * hwloc_dlhandle;
static __hwloc_inline int hwloc_dlinit(void) { return 0; }
static __hwloc_inline int hwloc_dlexit(void) { return 0; }
#define hwloc_dlclose dlclose
#define hwloc_dlerror dlerror
#define hwloc_dlsym dlsym

#include <sys/stat.h>
#include <sys/types.h>
#include <dirent.h>
#include <unistd.h>

static hwloc_dlhandle hwloc_dlopenext(const char *_filename)
{
  hwloc_dlhandle handle;
  char *filename = NULL;
  (void) asprintf(&filename, "%s.so", _filename);  # 将传入的文件名加上 .so 后缀
  if (!filename)
    return NULL;
  handle = dlopen(filename, RTLD_NOW|RTLD_LOCAL);  # 打开指定的动态链接库文件
  free(filename);  # 释放文件名内存
  return handle;  # 返回打开的动态链接库句柄
}

static int
hwloc_dlforeachfile(const char *_paths,
            int (*func)(const char *filename, void *data),
            void *data)
{
  char *paths = NULL, *path;

  paths = strdup(_paths);  # 复制传入的路径字符串
  if (!paths)
    return -1;

  path = paths;  # 指向复制的路径字符串
  while (*path) {  # 遍历路径字符串
    char *colon;
    DIR *dir;
    struct dirent *dirent;

    colon = strchr(path, ':');  # 查找路径中的冒号分隔符
    if (colon)
      *colon = '\0';  # 将冒号替换为字符串结束符

    if (hwloc_plugins_verbose)  # 如果插件加载时需要输出详细信息
      fprintf(stderr, "hwloc:  Looking under %s\n", path);  # 输出查找路径信息

    dir = opendir(path);  # 打开指定路径的目录
    if (!dir)  # 如果打开目录失败
      goto next;  # 跳过当前路径，继续下一个路径
    // 循环遍历目录，读取目录中的每个文件
    while ((dirent = readdir(dir)) != NULL) {
      // 定义变量
      char *abs_name, *suffix;
      struct stat stbuf;
      int err;

      // 根据目录和文件名生成绝对路径
      err = asprintf(&abs_name, "%s/%s", path, dirent->d_name);
      if (err < 0)
    continue;

      // 获取文件信息
      err = stat(abs_name, &stbuf);
      if (err < 0) {
    free(abs_name);
        continue;
      }
      // 如果不是普通文件，则跳过
      if (!S_ISREG(stbuf.st_mode)) {
    free(abs_name);
    continue;
      }

      /* 只保留 .so 文件，并去掉后缀以获取组件基本名称 */
      suffix = strrchr(abs_name, '.');
      if (!suffix || strcmp(suffix, ".so")) {
    free(abs_name);
    continue;
      }
      *suffix = '\0';

      // 调用函数处理文件
      err = func(abs_name, data);
      if (err) {
    free(abs_name);
    continue;
      }

      // 释放内存
      free(abs_name);
    }

    // 关闭目录
    closedir(dir);

  next:
    // 如果没有冒号，则跳出循环
    if (!colon)
      break;
    // 更新路径
    path = colon+1;
  }

  // 释放内存
  free(paths);
  // 返回 0 表示成功
  return 0;
/* 结束条件编译指令 */
#endif /* !HWLOC_HAVE_LTDL */

/* 指向动态加载插件的指针数组 */
static struct hwloc__plugin_desc {
  char *name;  // 插件名称
  struct hwloc_component *component;  // 插件组件结构体指针
  char *filename;  // 插件文件名
  hwloc_dlhandle handle;  // 插件句柄
  struct hwloc__plugin_desc *next;  // 指向下一个插件描述结构体的指针
} *hwloc_plugins = NULL;  // 初始化插件数组为NULL

static int
hwloc__dlforeach_cb(const char *filename, void *_data __hwloc_attribute_unused)
{
  const char *basename;  // 文件名的基本名称
  hwloc_dlhandle handle;  // 插件句柄
  struct hwloc_component *component;  // 插件组件结构体指针
  struct hwloc__plugin_desc *desc, **prevdesc;  // 插件描述结构体指针和指向前一个插件描述结构体指针的指针
  char *componentsymbolname;  // 组件符号名称

  if (hwloc_plugins_verbose)  // 如果插件详细信息开启
    fprintf(stderr, "hwloc: Plugin dlforeach found `%s'\n", filename);  // 输出找到的插件信息

  basename = strrchr(filename, '/');  // 获取文件名的基本名称
  if (!basename)  // 如果没有找到基本名称
    basename = filename;  // 则基本名称为整个文件名
  else  // 否则
    basename++;  // 基本名称为最后一个斜杠后的字符串

  if (hwloc_plugins_blacklist && strstr(hwloc_plugins_blacklist, basename)) {  // 如果插件黑名单存在且包含基本名称
    if (hwloc_plugins_verbose)  // 如果插件详细信息开启
      fprintf(stderr, "hwloc: Plugin `%s' is blacklisted in the environment\n", basename);  // 输出插件在环境中被列入黑名单的信息
    goto out;  // 跳转到结束标签
  }

  /* dlopen and get the component structure */
  handle = hwloc_dlopenext(filename);  // 打开插件文件
  if (!handle) {  // 如果打开失败
    if (hwloc_plugins_verbose)  // 如果插件详细信息开启
      fprintf(stderr, "hwloc: Failed to load plugin: %s\n", hwloc_dlerror());  // 输出加载插件失败的信息
    goto out;  // 跳转到结束标签
  }

  componentsymbolname = malloc(strlen(basename)+10+1);  // 分配组件符号名称的内存空间
  if (!componentsymbolname) {  // 如果分配失败
    if (hwloc_plugins_verbose)  // 如果插件详细信息开启
      fprintf(stderr, "hwloc: Failed to allocation component `%s' symbol\n",
          basename);  // 输出分配组件符号名称失败的信息
    goto out_with_handle;  // 跳转到带句柄结束标签
  }
  sprintf(componentsymbolname, "%s_component", basename);  // 格式化组件符号名称
  component = hwloc_dlsym(handle, componentsymbolname);  // 获取组件结构体指针
  if (!component) {  // 如果获取失败
    if (hwloc_plugins_verbose)  // 如果插件详细信息开启
      fprintf(stderr, "hwloc: Failed to find component symbol `%s'\n",
          componentsymbolname);  // 输出找不到组件符号名称的信息
    free(componentsymbolname);  // 释放组件符号名称的内存空间
    goto out_with_handle;  // 跳转到带句柄结束标签
  }
  if (component->abi != HWLOC_COMPONENT_ABI) {  // 如果组件符号名称的ABI与预期不符
    if (hwloc_plugins_verbose)  // 如果插件详细信息开启
      fprintf(stderr, "hwloc: Plugin symbol ABI %u instead of %d\n",
          component->abi, HWLOC_COMPONENT_ABI);  // 输出ABI不匹配的信息
    free(componentsymbolname);  // 释放组件符号名称的内存空间
    # 跳转到处理结束的标签
    goto out_with_handle;
  }
  # 如果插件包含预期的符号，则打印信息到标准错误输出
  if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Plugin contains expected symbol `%s'\n",
        componentsymbolname);
  # 释放组件符号名称的内存
  free(componentsymbolname);

  # 如果组件类型为发现类型
  if (HWLOC_COMPONENT_TYPE_DISC == component->type) {
    # 如果插件名称不符合发现类型的命名规范，则打印信息到标准错误输出，跳转到处理结束的标签
    if (strncmp(basename, "hwloc_", 6)) {
      if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Plugin name `%s' doesn't match its type DISCOVERY\n", basename);
      goto out_with_handle;
    }
  } else if (HWLOC_COMPONENT_TYPE_XML == component->type) {
    # 如果组件类型为 XML 类型
    # 如果插件名称不符合 XML 类型的命名规范，则打印信息到标准错误输出，跳转到处理结束的标签
    if (strncmp(basename, "hwloc_xml_", 10)) {
      if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Plugin name `%s' doesn't match its type XML\n", basename);
      goto out_with_handle;
    }
  } else {
    # 如果组件类型无效，则打印信息到标准错误输出，跳转到处理结束的标签
    if (hwloc_plugins_verbose)
      fprintf(stderr, "hwloc: Plugin name `%s' has invalid type %u\n",
          basename, (unsigned) component->type);
    goto out_with_handle;
  }

  # 分配一个插件描述符并将其加入队列
  desc = malloc(sizeof(*desc));
  # 如果分配内存失败，则跳转到处理结束的标签
  if (!desc)
    goto out_with_handle;
  # 复制插件名称和文件名到描述符中
  desc->name = strdup(basename);
  desc->filename = strdup(filename);
  desc->component = component;
  desc->handle = handle;
  desc->next = NULL;
  # 如果设置了插件详细信息输出，则打印信息到标准错误输出
  if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Plugin descriptor `%s' ready\n", basename);

  # 将描述符加入到列表中
  prevdesc = &hwloc_plugins;
  while (*prevdesc)
    prevdesc = &((*prevdesc)->next);
  *prevdesc = desc;
  # 如果设置了插件详细信息输出，则打印信息到标准错误输出
  if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Plugin descriptor `%s' queued\n", basename);
  # 返回 0 表示成功
  return 0;

 out_with_handle:
  # 关闭插件句柄
  hwloc_dlclose(handle);
 out:
  # 返回 0 表示成功
  return 0;
}

static void
hwloc_plugins_exit(void)
{
  struct hwloc__plugin_desc *desc, *next; // 定义指向插件描述结构体的指针变量desc和next

  if (hwloc_plugins_verbose) // 如果插件的详细信息标志为真
    fprintf(stderr, "hwloc: Closing all plugins\n"); // 输出关闭所有插件的信息到标准错误流

  desc = hwloc_plugins; // 将全局变量hwloc_plugins赋值给desc
  while (desc) { // 循环遍历插件描述结构体链表
    next = desc->next; // 将desc的下一个指针赋值给next
    hwloc_dlclose(desc->handle); // 关闭插件句柄
    free(desc->name); // 释放插件名称内存
    free(desc->filename); // 释放插件文件名内存
    free(desc); // 释放插件描述结构体内存
    desc = next; // 将next赋值给desc，继续下一个插件的处理
  }
  hwloc_plugins = NULL; // 将全局变量hwloc_plugins置空

  hwloc_dlexit(); // 调用插件动态链接库退出函数
}

static int
hwloc_plugins_init(void)
{
  const char *verboseenv; // 定义指向常量字符的指针变量verboseenv
  const char *path = HWLOC_PLUGINS_PATH; // 定义指向常量字符的指针变量path，并初始化为HWLOC_PLUGINS_PATH
  const char *env; // 定义指向常量字符的指针变量env
  int err; // 定义整型变量err

  verboseenv = getenv("HWLOC_PLUGINS_VERBOSE"); // 获取环境变量HWLOC_PLUGINS_VERBOSE的值赋给verboseenv
  hwloc_plugins_verbose = verboseenv ? atoi(verboseenv) : 0; // 如果verboseenv不为空，则将其转换为整型赋值给hwloc_plugins_verbose，否则赋值为0

  hwloc_plugins_blacklist = getenv("HWLOC_PLUGINS_BLACKLIST"); // 获取环境变量HWLOC_PLUGINS_BLACKLIST的值赋给hwloc_plugins_blacklist

  err = hwloc_dlinit(); // 调用插件动态链接库初始化函数
  if (err) // 如果初始化失败
    goto out; // 跳转到标签out

  env = getenv("HWLOC_PLUGINS_PATH"); // 获取环境变量HWLOC_PLUGINS_PATH的值赋给env
  if (env) // 如果env不为空
    path = env; // 将env赋值给path

  hwloc_plugins = NULL; // 将全局变量hwloc_plugins置空

  if (hwloc_plugins_verbose) // 如果插件的详细信息标志为真
    fprintf(stderr, "hwloc: Starting plugin dlforeach in %s\n", path); // 输出插件dlforeach的启动信息到标准错误流
  err = hwloc_dlforeachfile(path, hwloc__dlforeach_cb, NULL); // 调用插件动态链接库遍历文件函数
  if (err) // 如果遍历文件失败
    goto out_with_init; // 跳转到标签out_with_init

  return 0; // 返回0，表示初始化成功

 out_with_init:
  hwloc_plugins_exit(); // 调用插件退出函数
 out:
  return -1; // 返回-1，表示初始化失败
}

#endif /* HWLOC_HAVE_PLUGINS */

static int
hwloc_disc_component_register(struct hwloc_disc_component *component,
                  const char *filename)
{
  struct hwloc_disc_component **prev; // 定义指向指向发现组件结构体指针的指针变量prev

  /* check that the component name is valid */
  if (!strcmp(component->name, HWLOC_COMPONENT_STOP_NAME)) { // 如果组件名称与停止名称相同
    if (hwloc_components_verbose) // 如果组件的详细信息标志为真
      fprintf(stderr, "hwloc: Cannot register discovery component with reserved name `" HWLOC_COMPONENT_STOP_NAME "'\n"); // 输出无法注册具有保留名称的发现组件的信息到标准错误流
    return -1; // 返回-1，表示注册失败
  }
  if (strchr(component->name, HWLOC_COMPONENT_EXCLUDE_CHAR) // 如果组件名称中包含排除字符
      || strchr(component->name, HWLOC_COMPONENT_PHASESEP_CHAR) // 或者包含阶段分隔字符
      || strcspn(component->name, HWLOC_COMPONENT_SEPS) != strlen(component->name)) { // 或者组件名称中不包含分隔字符
    # 如果设置了硬件位置组件的详细输出
    if (hwloc_components_verbose)
      # 输出错误信息，指出无法注册包含保留字符的发现组件名称
      fprintf(stderr, "hwloc: Cannot register discovery component with name `%s' containing reserved characters `%c" HWLOC_COMPONENT_SEPS "'\n",
          component->name, HWLOC_COMPONENT_EXCLUDE_CHAR);
    # 返回错误代码
    return -1;
  }

  # 检查组件阶段是否有效
  if (!component->phases
      || (component->phases != HWLOC_DISC_PHASE_GLOBAL
      && component->phases & ~(HWLOC_DISC_PHASE_CPU
                   |HWLOC_DISC_PHASE_MEMORY
                   |HWLOC_DISC_PHASE_PCI
                   |HWLOC_DISC_PHASE_IO
                   |HWLOC_DISC_PHASE_MISC
                   |HWLOC_DISC_PHASE_ANNOTATE
                   |HWLOC_DISC_PHASE_TWEAK))) {
    # 如果设置了关键错误的显示
    if (HWLOC_SHOW_CRITICAL_ERRORS())
      # 输出错误信息，指出无法注册包含无效阶段的发现组件
      fprintf(stderr, "hwloc: Cannot register discovery component `%s' with invalid phases 0x%x\n",
              component->name, component->phases);
    # 返回错误代码
    return -1;
  }

  # 遍历已注册的硬件位置发现组件
  prev = &hwloc_disc_components;
  while (NULL != *prev) {
    # 如果发现两个组件具有相同的名称
    if (!strcmp((*prev)->name, component->name)) {
      # 如果两个组件具有相同的名称，则只保留优先级较高的组件
      if ((*prev)->priority < component->priority) {
    # 丢弃现有组件
    if (hwloc_components_verbose)
      # 输出警告信息，指出较低优先级的组件被丢弃
      fprintf(stderr, "hwloc: Dropping previously registered discovery component `%s', priority %u lower than new one %u\n",
          (*prev)->name, (*prev)->priority, component->priority);
    *prev = (*prev)->next;
      } else {
    # 丢弃新组件
    if (hwloc_components_verbose)
      # 输出警告信息，指出较低优先级的新组件被忽略
      fprintf(stderr, "hwloc: Ignoring new discovery component `%s', priority %u lower than previously registered one %u\n",
          component->name, component->priority, (*prev)->priority);
    # 返回错误代码
    return -1;
      }
    }
    # 继续遍历下一个组件
    prev = &((*prev)->next);
  }
  # 如果设置了硬件位置组件的详细输出
  if (hwloc_components_verbose)
    # 打印注册的发现组件信息，包括组件名称、阶段、优先级和来源
    fprintf(stderr, "hwloc: Registered discovery component `%s' phases 0x%x with priority %u (%s%s)\n",
        component->name, component->phases, component->priority,
        filename ? "from plugin " : "statically build", filename ? filename : "");

  # 设置指向发现组件链表的指针
  prev = &hwloc_disc_components;
  # 遍历发现组件链表，找到优先级比当前组件低的位置
  while (NULL != *prev) {
    if ((*prev)->priority < component->priority)
      break;
    prev = &((*prev)->next);
  }
  # 将当前组件插入到链表中
  component->next = *prev;
  *prev = component;
  # 返回成功
  return 0;
}

#include "static-components.h"  // 包含静态组件的头文件

static void (**hwloc_component_finalize_cbs)(unsigned long);  // 定义指向无返回值函数指针的指针
static unsigned hwloc_component_finalize_cb_count;  // 定义无符号整数类型的变量

void
hwloc_components_init(void)  // 初始化硬件位置组件
{
#ifdef HWLOC_HAVE_PLUGINS  // 如果定义了 HWLOC_HAVE_PLUGINS
  struct hwloc__plugin_desc *desc;  // 定义 hwloc__plugin_desc 结构体指针 desc
#endif
  const char *verboseenv;  // 定义指向字符常量的指针
  unsigned i;  // 定义无符号整数类型的变量

  HWLOC_COMPONENTS_LOCK();  // 锁定硬件位置组件
  assert((unsigned) -1 != hwloc_components_users);  // 断言确保 hwloc_components_users 不等于 -1
  if (0 != hwloc_components_users++) {  // 如果 hwloc_components_users 不等于 0
    HWLOC_COMPONENTS_UNLOCK();  // 解锁硬件位置组件
    return;  // 返回
  }

  verboseenv = getenv("HWLOC_COMPONENTS_VERBOSE");  // 获取环境变量 HWLOC_COMPONENTS_VERBOSE 的值
  hwloc_components_verbose = verboseenv ? atoi(verboseenv) : 0;  // 如果 verboseenv 不为空，则将其转换为整数，否则为 0

#ifdef HWLOC_HAVE_PLUGINS  // 如果定义了 HWLOC_HAVE_PLUGINS
  hwloc_plugins_init();  // 初始化硬件位置插件
#endif

  hwloc_component_finalize_cbs = NULL;  // 将 hwloc_component_finalize_cbs 设置为 NULL
  hwloc_component_finalize_cb_count = 0;  // 将 hwloc_component_finalize_cb_count 设置为 0
  /* count the max number of finalize callbacks */
  for(i=0; NULL != hwloc_static_components[i]; i++)  // 遍历静态组件
    hwloc_component_finalize_cb_count++;  // 计算最大的 finalize 回调数量
#ifdef HWLOC_HAVE_PLUGINS  // 如果定义了 HWLOC_HAVE_PLUGINS
  for(desc = hwloc_plugins; NULL != desc; desc = desc->next)  // 遍历硬件位置插件
    hwloc_component_finalize_cb_count++;  // 计算最大的 finalize 回调数量
#endif
  if (hwloc_component_finalize_cb_count) {  // 如果 finalize 回调数量不为 0
    hwloc_component_finalize_cbs = calloc(hwloc_component_finalize_cb_count,  // 分配内存给 hwloc_component_finalize_cbs
                      sizeof(*hwloc_component_finalize_cbs));
    assert(hwloc_component_finalize_cbs);  // 断言确保分配内存成功
    /* forget that max number and recompute the real one below */
    hwloc_component_finalize_cb_count = 0;  // 将 finalize 回调数量重新设置为 0
  }

  /* hwloc_static_components is created by configure in static-components.h */
  for(i=0; NULL != hwloc_static_components[i]; i++) {  // 遍历静态组件
    if (hwloc_static_components[i]->flags) {  // 如果静态组件的标志不为 0
      if (HWLOC_SHOW_CRITICAL_ERRORS())  // 如果显示关键错误
        fprintf(stderr, "hwloc: Ignoring static component with invalid flags %lx\n",
                hwloc_static_components[i]->flags);  // 输出错误信息
      continue;  // 继续下一次循环
    }

    /* initialize the component */
    if (hwloc_static_components[i]->init && hwloc_static_components[i]->init(0) < 0) {  // 如果静态组件的初始化函数存在且初始化失败
      if (hwloc_components_verbose)  // 如果硬件位置组件详细信息为真
    fprintf(stderr, "hwloc: Ignoring static component, failed to initialize\n");  // 输出错误信息
      continue;  // 继续下一次循环
    }
    /* 如果存在队列中的组件需要执行 finalize() 回调函数，则将其添加到回调函数数组中 */
    if (hwloc_static_components[i]->finalize)
      hwloc_component_finalize_cbs[hwloc_component_finalize_cb_count++] = hwloc_static_components[i]->finalize;

    /* 真正注册组件 */
    if (HWLOC_COMPONENT_TYPE_DISC == hwloc_static_components[i]->type)
      hwloc_disc_component_register(hwloc_static_components[i]->data, NULL);
    else if (HWLOC_COMPONENT_TYPE_XML == hwloc_static_components[i]->type)
      hwloc_xml_callbacks_register(hwloc_static_components[i]->data);
    else
      assert(0);
  }

  /* 动态插件 */
#ifdef HWLOC_HAVE_PLUGINS
  // 如果支持插件
  for(desc = hwloc_plugins; NULL != desc; desc = desc->next) {
    // 遍历插件列表
    if (desc->component->flags) {
      // 如果插件组件的标志位不为0
      if (HWLOC_SHOW_CRITICAL_ERRORS())
        // 如果需要显示关键错误
        fprintf(stderr, "hwloc: Ignoring plugin `%s' component with invalid flags %lx\n",
                desc->name, desc->component->flags);
      // 继续下一个插件
      continue;
    }

    /* initialize the component */
    // 初始化组件
    if (desc->component->init && desc->component->init(0) < 0) {
      // 如果初始化失败
      if (hwloc_components_verbose)
        // 如果需要显示详细信息
        fprintf(stderr, "hwloc: Ignoring plugin `%s', failed to initialize\n", desc->name);
      // 继续下一个插件
      continue;
    }
    /* queue ->finalize() callback if any */
    // 如果存在 finalize() 回调函数，则加入队列
    if (desc->component->finalize)
      hwloc_component_finalize_cbs[hwloc_component_finalize_cb_count++] = desc->component->finalize;

    /* register for real now */
    // 真正注册组件
    if (HWLOC_COMPONENT_TYPE_DISC == desc->component->type)
      // 如果是发现组件
      hwloc_disc_component_register(desc->component->data, desc->filename);
    else if (HWLOC_COMPONENT_TYPE_XML == desc->component->type)
      // 如果是 XML 组件
      hwloc_xml_callbacks_register(desc->component->data);
    else
      // 否则断言失败
      assert(0);
  }
#endif

  // 解锁组件
  HWLOC_COMPONENTS_UNLOCK();
}

void
hwloc_topology_components_init(struct hwloc_topology *topology)
{
  // 初始化拓扑结构的组件信息
  topology->nr_blacklisted_components = 0;
  topology->blacklisted_components = NULL;

  topology->backends = NULL;
  topology->backend_phases = 0;
  topology->backend_excluded_phases = 0;
}

/* look for name among components, ignoring things after `:' */
// 在组件中查找名称，忽略冒号后的内容
static struct hwloc_disc_component *
hwloc_disc_component_find(const char *name, const char **endp)
{
  // 查找组件
  struct hwloc_disc_component *comp;
  size_t length;
  const char *end = strchr(name, HWLOC_COMPONENT_PHASESEP_CHAR);
  if (end) {
    length = end-name;
    if (endp)
      *endp = end+1;
  } else {
    length = strlen(name);
    if (endp)
      *endp = NULL;
  }

  comp = hwloc_disc_components;
  while (NULL != comp) {
    if (!strncmp(name, comp->name, length))
      return comp;
    comp = comp->next;
  }
  return NULL;
}

static unsigned
// 从字符串中解析出硬件定位的阶段
hwloc_phases_from_string(const char *s)
{
  // 如果字符串为空，则返回最大无符号整数
  if (!s)
    return ~0U;
  // 如果字符串的第一个字符不是数字，则根据字符串内容返回对应的硬件定位阶段
  if (s[0]<'0' || s[0]>'9') {
    if (!strcasecmp(s, "global"))
      return HWLOC_DISC_PHASE_GLOBAL;
    else if (!strcasecmp(s, "cpu"))
      return HWLOC_DISC_PHASE_CPU;
    if (!strcasecmp(s, "memory"))
      return HWLOC_DISC_PHASE_MEMORY;
    if (!strcasecmp(s, "pci"))
      return HWLOC_DISC_PHASE_PCI;
    if (!strcasecmp(s, "io"))
      return HWLOC_DISC_PHASE_IO;
    if (!strcasecmp(s, "misc"))
      return HWLOC_DISC_PHASE_MISC;
    if (!strcasecmp(s, "annotate"))
      return HWLOC_DISC_PHASE_ANNOTATE;
    if (!strcasecmp(s, "tweak"))
      return HWLOC_DISC_PHASE_TWEAK;
    return 0;
  }
  // 如果字符串的第一个字符是数字，则将字符串转换为无符号整数
  return (unsigned) strtoul(s, NULL, 0);
}

// 黑名单一个硬件定位组件
static int
hwloc_disc_component_blacklist_one(struct hwloc_topology *topology,
                   const char *name)
{
  struct hwloc_topology_forced_component_s *blacklisted;
  struct hwloc_disc_component *comp;
  unsigned phases;
  unsigned i;

  // 如果组件名为 linuxpci 或 linuxio，则替换为 linux，并设置对应的硬件定位阶段
  if (!strcmp(name, "linuxpci") || !strcmp(name, "linuxio")) {
    if (hwloc_components_verbose)
      fprintf(stderr, "hwloc: Replacing deprecated component `%s' with `linux' IO phases in blacklisting\n", name);
    comp = hwloc_disc_component_find("linux", NULL);
    phases = HWLOC_DISC_PHASE_PCI | HWLOC_DISC_PHASE_IO | HWLOC_DISC_PHASE_MISC | HWLOC_DISC_PHASE_ANNOTATE;

  } else {
    // 否则进行正常的查找
    const char *end;
    comp = hwloc_disc_component_find(name, &end);
    phases = hwloc_phases_from_string(end);
  }
  // 如果未找到组件，则设置错误码并返回 -1
  if (!comp) {
    errno = EINVAL;
    return -1;
  }

  // 如果组件查找过程中启用了详细输出，则打印黑名单组件的信息
  if (hwloc_components_verbose)
    fprintf(stderr, "hwloc: Blacklisting component `%s` phases 0x%x\n", comp->name, phases);

  // 遍历黑名单组件列表
  for(i=0; i<topology->nr_blacklisted_components; i++) {
    # 如果当前组件已经在黑名单中，则更新其相应的相位信息并返回0
    if (topology->blacklisted_components[i].component == comp) {
      topology->blacklisted_components[i].phases |= phases;
      return 0;
    }
  }

  # 重新分配内存以扩展黑名单组件数组
  blacklisted = realloc(topology->blacklisted_components, (topology->nr_blacklisted_components+1)*sizeof(*blacklisted));
  # 如果内存分配失败，则返回-1
  if (!blacklisted)
    return -1;

  # 将新的组件和相位信息添加到黑名单数组中
  blacklisted[topology->nr_blacklisted_components].component = comp;
  blacklisted[topology->nr_blacklisted_components].phases = phases;
  # 更新黑名单组件数组的指针
  topology->blacklisted_components = blacklisted;
  # 增加黑名单组件数量
  topology->nr_blacklisted_components++;
  # 返回0表示成功
  return 0;
# 设置硬件拓扑结构的组件
int
hwloc_topology_set_components(struct hwloc_topology *topology,
                  unsigned long flags,
                  const char *name)
{
  # 如果拓扑结构已经加载，则返回忙碌错误
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  # 如果标志位包含非法标志，则返回无效参数错误
  if (flags & ~HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST) {
    errno = EINVAL;
    return -1;
  }

  # 目前这个标志位是必需的
  if (flags != HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST) {
    errno = EINVAL;
    return -1;
  }

  # 如果名称以 "all" 开头并且后面紧跟 HWLOC_COMPONENT_PHASESEP_CHAR，则设置拓扑结构的后端排除阶段
  if (!strncmp(name, "all", 3) && name[3] == HWLOC_COMPONENT_PHASESEP_CHAR) {
    topology->backend_excluded_phases = hwloc_phases_from_string(name+4);
    return 0;
  }

  # 调用函数设置拓扑结构的组件黑名单
  return hwloc_disc_component_blacklist_one(topology, name);
}

# 用于强制启用第一个后端，被 set_xml()、set_synthetic() 等环境变量调用
int
hwloc_disc_component_force_enable(struct hwloc_topology *topology,
                  int envvar_forced,
                  const char *name,
                  const void *data1, const void *data2, const void *data3)
{
  struct hwloc_disc_component *comp;
  struct hwloc_backend *backend;

  # 如果拓扑结构已经加载，则返回忙碌错误
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  # 查找指定名称的组件，如果找不到则返回不支持错误
  comp = hwloc_disc_component_find(name, NULL);
  if (!comp) {
    errno = ENOSYS;
    return -1;
  }

  # 实例化组件，强制启用，获取后端
  backend = comp->instantiate(topology, comp, 0U /* force-enabled don't get any phase blacklisting */,
                  data1, data2, data3);
  if (backend) {
    int err;
    backend->envvar_forced = envvar_forced;
    if (topology->backends)
      hwloc_backends_disable_all(topology);
    err = hwloc_backend_enable(backend);

    # 如果组件的阶段为全局阶段，则根据环境变量设置后端排除阶段
    if (comp->phases == HWLOC_DISC_PHASE_GLOBAL) {
      char *env = getenv("HWLOC_ANNOTATE_GLOBAL_COMPONENTS");
      if (env && atoi(env))
    topology->backend_excluded_phases &= ~HWLOC_DISC_PHASE_ANNOTATE;
    }

    return err;
  } else
    return -1;
}

static int
hwloc_disc_component_try_enable(struct hwloc_topology *topology,
                struct hwloc_disc_component *comp,
                int envvar_forced,
                unsigned blacklisted_phases)
{
  struct hwloc_backend *backend;

  // 检查组件的探测阶段是否已经被排除，如果是，则完全排除该组件
  if (!(comp->phases & ~(topology->backend_excluded_phases | blacklisted_phases))) {
    // 如果环境变量强制设置了，则不发出警告
    if (hwloc_components_verbose)
      fprintf(stderr, "hwloc: Excluding discovery component `%s' phases 0x%x, conflicts with excludes 0x%x\n",
          comp->name, comp->phases, topology->backend_excluded_phases);
    return -1;
  }

  // 实例化组件，获取对应的后端对象
  backend = comp->instantiate(topology, comp, topology->backend_excluded_phases | blacklisted_phases,
                  NULL, NULL, NULL);
  // 如果实例化失败，则根据环境变量强制设置和关键错误的条件决定是否发出警告
  if (!backend) {
    if (hwloc_components_verbose || (envvar_forced && HWLOC_SHOW_CRITICAL_ERRORS()))
      fprintf(stderr, "hwloc: Failed to instantiate discovery component `%s'\n", comp->name);
    return -1;
  }

  // 排除已经被列入黑名单的探测阶段
  backend->phases &= ~blacklisted_phases;
  backend->envvar_forced = envvar_forced;
  // 启用后端对象
  return hwloc_backend_enable(backend);
}

void
hwloc_disc_components_enable_others(struct hwloc_topology *topology)
{
  struct hwloc_disc_component *comp;
  struct hwloc_backend *backend;
  int tryall = 1;
  const char *_env;
  char *env; // 需要修改 env 值，因此复制一份
  unsigned i;

  _env = getenv("HWLOC_COMPONENTS");
  env = _env ? strdup(_env) : NULL;

  // 黑名单禁用的组件
  if (env) {
    char *curenv = env;
    size_t s;

    while (*curenv) {
      s = strcspn(curenv, HWLOC_COMPONENT_SEPS);
      if (s) {
    char c;

    if (curenv[0] != HWLOC_COMPONENT_EXCLUDE_CHAR)
      goto nextname;

    // 保存最后一个字符并替换为 \0
    c = curenv[s];
    curenv[s] = '\0';
    /* 将当前组件加入黑名单，并忽略分配失败的情况 */
    hwloc_disc_component_blacklist_one(topology, curenv+1);

    /* 从字符串中移除被加入黑名单的组件名 */
    for(i=0; i<s; i++)
      curenv[i] = *HWLOC_COMPONENT_SEPS;

    /* 恢复字符（下面的循环需要 env 变量未被修改） */
    curenv[s] = c;
      }

    nextname:
      curenv += s;
      if (*curenv)
    /* 跳过逗号 */
    curenv++;
    }
  }

  /* 显式启用列出的组件 */
  if (env) {
    char *curenv = env;
    size_t s;

    while (*curenv) {
      s = strcspn(curenv, HWLOC_COMPONENT_SEPS);
      if (s) {
    char c;
    const char *name;

    if (!strncmp(curenv, HWLOC_COMPONENT_STOP_NAME, s)) {
      tryall = 0;
      break;
    }

    /* 保存最后一个字符并替换为 \0 */
    c = curenv[s];
    curenv[s] = '\0';

    name = curenv;
    if (!strcmp(name, "linuxpci") || !strcmp(name, "linuxio")) {
      if (hwloc_components_verbose)
        fprintf(stderr, "hwloc: Replacing deprecated component `%s' with `linux' in envvar forcing\n", name);
      name = "linux";
    }

    comp = hwloc_disc_component_find(name, NULL /* 我们启用整个组件，阶段必须单独加入黑名单 */);
    if (comp) {
      unsigned blacklisted_phases = 0U;
      for(i=0; i<topology->nr_blacklisted_components; i++)
        if (comp == topology->blacklisted_components[i].component) {
          blacklisted_phases = topology->blacklisted_components[i].phases;
          break;
        }
      if (comp->phases & ~blacklisted_phases)
        hwloc_disc_component_try_enable(topology, comp, 1 /* envvar 强制 */, blacklisted_phases);
    } else {
          if (HWLOC_SHOW_CRITICAL_ERRORS())
            fprintf(stderr, "hwloc: Cannot find discovery component `%s'\n", name);
    }

    /* 恢复字符（下面的循环需要 env 变量未被修改） */
    curenv[s] = c;
      }

      curenv += s;
      if (*curenv)
    /* 跳过逗号 */
    curenv++;
  }
}

/* 环境变量仍然是相同的，上面的循环没有修改它 */

/* 现在启用剩余的组件（除了明确列出的“-”） */
if (tryall) {
  comp = hwloc_disc_components;
  while (NULL != comp) {
    unsigned blacklisted_phases = 0U;
    if (!comp->enabled_by_default)
      goto nextcomp;
    /* 检查应用程序是否将此组件列入黑名单 */
    for(i=0; i<topology->nr_blacklisted_components; i++)
      if (comp == topology->blacklisted_components[i].component) {
        blacklisted_phases = topology->blacklisted_components[i].phases;
        break;
      }

    if (!(comp->phases & ~blacklisted_phases)) {
      if (hwloc_components_verbose)
        fprintf(stderr, "hwloc: Excluding blacklisted discovery component `%s' phases 0x%x\n",
            comp->name, comp->phases);
      goto nextcomp;
    }

    hwloc_disc_component_try_enable(topology, comp, 0 /* defaults, not envvar forced */, blacklisted_phases);
void
hwloc_components_fini(void)
{
  unsigned i;

  // 加锁，确保在多线程环境下操作的原子性
  HWLOC_COMPONENTS_LOCK();
  // 断言组件的使用次数不为0
  assert(0 != hwloc_components_users);
  // 如果组件的使用次数不为0，则解锁并返回
  if (0 != --hwloc_components_users) {
    HWLOC_COMPONENTS_UNLOCK();
    return;
  }

  // 循环调用组件的终结函数
  for(i=0; i<hwloc_component_finalize_cb_count; i++)
    hwloc_component_finalize_cbs[hwloc_component_finalize_cb_count-i-1](0);
  // 释放组件终结函数的内存
  free(hwloc_component_finalize_cbs);
  hwloc_component_finalize_cbs = NULL;
  hwloc_component_finalize_cb_count = 0;

  // 重置组件列表
  // 不需要解除链接/释放组件列表，它们将在下面被卸载
  hwloc_disc_components = NULL;
  // 重置 XML 回调
  hwloc_xml_callbacks_reset();

  // 如果支持插件，则退出插件
#ifdef HWLOC_HAVE_PLUGINS
  hwloc_plugins_exit();
#endif

  // 解锁
  HWLOC_COMPONENTS_UNLOCK();
}

// 分配一个后端对象
struct hwloc_backend *
hwloc_backend_alloc(struct hwloc_topology *topology,
            struct hwloc_disc_component *component)
{
  // 分配后端对象的内存
  struct hwloc_backend * backend = malloc(sizeof(*backend));
  // 如果内存分配失败，则设置错误码并返回空指针
  if (!backend) {
    errno = ENOMEM;
    return NULL;
  }
  // 设置后端对象的组件和拓扑
  backend->component = component;
  backend->topology = topology;
  // 过滤掉被排除的组件阶段
  backend->phases = component->phases & ~topology->backend_excluded_phases;
  // 如果被排除的组件阶段不等于组件的阶段，并且组件的使用次数为0，则打印详细信息
  if (backend->phases != component->phases && hwloc_components_verbose)
    # 输出错误信息到标准错误流，显示尝试使用的发现组件名称、使用的阶段和组件定义的阶段
    fprintf(stderr, "hwloc: Trying discovery component `%s' with phases 0x%x instead of 0x%x\n",
        component->name, backend->phases, component->phases);
  # 将后端标志位设置为0
  backend->flags = 0;
  # 将后端的发现函数指针设置为NULL
  backend->discover = NULL;
  # 将后端的获取PCI总线ID的CPU集合函数指针设置为NULL
  backend->get_pci_busid_cpuset = NULL;
  # 将后端的禁用函数指针设置为NULL
  backend->disable = NULL;
  # 将后端的是否为本系统的标志位设置为-1
  backend->is_thissystem = -1;
  # 将后端的下一个指针设置为NULL
  backend->next = NULL;
  # 将后端的环境变量强制标志位设置为0
  backend->envvar_forced = 0;
  # 返回后端对象
  return backend;
# 禁用指定的后端
static void
hwloc_backend_disable(struct hwloc_backend *backend)
{
  # 如果存在禁用函数，则调用该函数
  if (backend->disable)
    backend->disable(backend);
  # 释放后端对象的内存
  free(backend);
}

# 启用指定的后端
int
hwloc_backend_enable(struct hwloc_backend *backend)
{
  # 获取拓扑结构
  struct hwloc_topology *topology = backend->topology;
  struct hwloc_backend **pprev;

  # 检查后端标志
  if (backend->flags) {
    # 如果存在关键错误，则打印错误信息
    if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Cannot enable discovery component `%s' phases 0x%x with unknown flags %lx\n",
              backend->component->name, backend->component->phases, backend->flags);
    return -1;
  }

  # 确保没有重复启用相同的后端
  pprev = &topology->backends;
  while (NULL != *pprev) {
    if ((*pprev)->component == backend->component) {
      # 如果启用了相同的后端，则打印错误信息
      if (hwloc_components_verbose)
    fprintf(stderr, "hwloc: Cannot enable  discovery component `%s' phases 0x%x twice\n",
        backend->component->name, backend->component->phases);
      # 禁用后端
      hwloc_backend_disable(backend);
      errno = EBUSY;
      return -1;
    }
    pprev = &((*pprev)->next);
  }

  # 如果启用了相同的后端，则打印启用信息
  if (hwloc_components_verbose)
    fprintf(stderr, "hwloc: Enabling discovery component `%s' with phases 0x%x (among 0x%x)\n",
        backend->component->name, backend->phases, backend->component->phases);

  # 将后端加入到拓扑结构的后端列表中
  pprev = &topology->backends;
  while (NULL != *pprev)
    pprev = &((*pprev)->next);
  backend->next = *pprev;
  *pprev = backend;

  # 更新拓扑结构的后端阶段和排除阶段
  topology->backend_phases |= backend->component->phases;
  topology->backend_excluded_phases |= backend->component->excluded_phases;
  return 0;
}

# 判断后端是否为当前系统
void
hwloc_backends_is_thissystem(struct hwloc_topology *topology)
  // 声明指向 hwloc_backend 结构体的指针变量
  struct hwloc_backend *backend;
  // 声明指向常量字符的指针变量
  const char *local_env;

  /*
   * 如果应用程序使用 set_foo() 改变了后端，
   * 可能会使用 set_flags() 在这里更新 is_thissystem 标志。
   * 如果使用下面的环境变量改变了后端，
   * 也可以使用下面的 HWLOC_THISSYSTEM 环境变量。
   */

  // 将 topology 结构体中的 is_thissystem 标志设置为 1
  topology->is_thissystem = 1;

  /* 应用来自通常给定的后端的 thissystem（envvar_forced=0，要么是 set_foo() 要么是默认值） */
  // 遍历后端链表
  backend = topology->backends;
  while (backend != NULL) {
    // 如果后端的 envvar_forced 为 0，并且 is_thissystem 不为 -1
    if (backend->envvar_forced == 0 && backend->is_thissystem != -1) {
      // 断言后端的 is_thissystem 为 0
      assert(backend->is_thissystem == 0);
      // 将 topology 结构体中的 is_thissystem 标志设置为 0
      topology->is_thissystem = 0;
    }
    // 移动到下一个后端
    backend = backend->next;
  }

  /* 使用 flags 覆盖 set_foo() */
  // 如果 topology 结构体中的 flags 包含 HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM 标志
  if (topology->flags & HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM)
    // 将 topology 结构体中的 is_thissystem 标志设置为 1
    topology->is_thissystem = 1;

  /* 现在应用 envvar-forced 后端（envvar_forced=1） */
  // 重新遍历后端链表
  backend = topology->backends;
  while (backend != NULL) {
    // 如果后端的 envvar_forced 为 1，并且 is_thissystem 不为 -1
    if (backend->envvar_forced == 1 && backend->is_thissystem != -1) {
      // 断言后端的 is_thissystem 为 0
      assert(backend->is_thissystem == 0);
      // 将 topology 结构体中的 is_thissystem 标志设置为 0
      topology->is_thissystem = 0;
    }
    // 移动到下一个后端
    backend = backend->next;
  }

  /* 使用环境变量给定的标志进行覆盖 */
  // 从环境变量中获取 HWLOC_THISSYSTEM 的值
  local_env = getenv("HWLOC_THISSYSTEM");
  // 如果 local_env 不为空
  if (local_env)
    // 将 topology 结构体中的 is_thissystem 标志设置为 local_env 的整数值
    topology->is_thissystem = atoi(local_env);
}

// 查找后端的回调函数
void
hwloc_backends_find_callbacks(struct hwloc_topology *topology)
{
  // 声明指向 hwloc_backend 结构体的指针变量
  struct hwloc_backend *backend = topology->backends;
  // 使用第一个后端的 get_pci_busid_cpuset 回调函数
  topology->get_pci_busid_cpuset_backend = NULL;
  // 遍历后端链表
  while (backend != NULL) {
    // 如果后端的 get_pci_busid_cpuset 不为空
    if (backend->get_pci_busid_cpuset) {
      // 将 topology 结构体中的 get_pci_busid_cpuset_backend 指向当前后端
      topology->get_pci_busid_cpuset_backend = backend;
      // 返回
      return;
    }
    // 移动到下一个后端
    backend = backend->next;
  }
  // 返回
  return;
}

// 禁用所有后端
void
hwloc_backends_disable_all(struct hwloc_topology *topology)
{
  // 声明指向 hwloc_backend 结构体的指针变量
  struct hwloc_backend *backend;

  // 循环直到后端链表为空
  while (NULL != (backend = topology->backends)) {
    // 保存下一个后端
    struct hwloc_backend *next = backend->next;
    # 如果 hwloc_components_verbose 为真，则向标准错误流输出信息，指明禁用的发现组件名称
    if (hwloc_components_verbose)
      fprintf(stderr, "hwloc: Disabling discovery component `%s'\n",
          backend->component->name);
    # 禁用当前后端
    hwloc_backend_disable(backend);
    # 将下一个后端设置为当前拓扑结构的后端
    topology->backends = next;
  }
  # 将拓扑结构的后端设置为空
  topology->backends = NULL;
  # 将拓扑结构的后端排除的阶段数设置为0
  topology->backend_excluded_phases = 0;
# 结束 hwloc_topology_components_fini 函数

void
hwloc_topology_components_fini(struct hwloc_topology *topology)
{
  # 断言确保在调用该函数之前已经调用了 hwloc_backends_disable_all() 函数
  assert(!topology->backends);

  # 释放黑名单组件的内存空间
  free(topology->blacklisted_components);
}
```