# xmrig源码解析 31

# `src/3rdparty/hwloc/src/components.c`

这段代码定义了一个名为 "autogen" 的私有组件，并使用了几个自定义的函数，如 xml_parse() 和 config_read_dir()。它还包含了硬编码的停止名称、排除字符串和组件序列化格式。

具体来说，这个组件的功能是用于配置文件中定义的硬件组件。它通过 xml_parse() 函数读取配置文件，并使用 config_read_dir() 函数读取目录列表中的硬件组件配置。在读取完配置文件后，它使用 xml_format() 函数将硬件组件配置格式化为包含 "stop" 字样的字符串，并将结果保存到 HWLOC_COMPONENT_STOP_NAME 常量中。


```cpp
/*
 * Copyright © 2009-2022 Inria.  All rights reserved.
 * Copyright © 2012 Université Bordeaux
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "private/xml.h"
#include "private/misc.h"

#define HWLOC_COMPONENT_STOP_NAME "stop"
#define HWLOC_COMPONENT_EXCLUDE_CHAR '-'
#define HWLOC_COMPONENT_SEPS ","
```

这段代码定义了一个名为 `HWLOC_COMPONENT_PHASESEP_CHAR` 的宏，表示为组件 phase separateChar。接下来定义了一个列表，列出了所有注册的发现组件，按照优先级排序，优先级越高在列表中越靠前。其中，NoS分类别为最低优先级，因为它没有对应的插件。其他的组件则有对应的插件，如 `hwloc_disc_component`，它们的优先级比 `NoS` 更高。

然后定义了一个名为 `hwloc_disc_components` 的指向整型数组的指针，用于存储所有注册的组件。接下来定义了一个名为 `hwloc_components_users` 的整型变量，用于存储注册组件的用户数。

最后定义了一个名为 `hwloc_components_verbose` 的整型变量，用于存储是否输出组件注册信息中的 `verbose` 标志。如果这个标志为 1，则会输出更多的信息。

整段代码的作用是定义了 `HWLOC_COMPONENT_PHASESEP_CHAR` 这个宏，以及它所代表的意义。


```cpp
#define HWLOC_COMPONENT_PHASESEP_CHAR ':'

/* list of all registered discovery components, sorted by priority, higher priority first.
 * noos is last because its priority is 0.
 * others' priority is 10.
 */
static struct hwloc_disc_component * hwloc_disc_components = NULL;

static unsigned hwloc_components_users = 0; /* first one initializes, last ones destroys */

static int hwloc_components_verbose = 0;
#ifdef HWLOC_HAVE_PLUGINS
static int hwloc_plugins_verbose = 0;
static const char * hwloc_plugins_blacklist = NULL;
#endif

```

这段代码定义了一个名为 `hwloc_components_mutex` 的互斥锁，用于在 hwloc_plugins 加载和修改过程中，确保只有一个进程在执行注册和卸载组件等操作。

该互斥锁通过在 `hwloc_check_plugin_namespace()` 函数中调用 `ltdl`（一个用于加载动态模块的库）来实现。在 `hwloc_xml_callbacks_register()` 函数中，该互斥锁还用于注册 HWLOC_PLUGIN_HANDLE_FUNCTION() 函数指针，以便在注册组件时自动调用该函数。

该代码中，互斥锁 `hwloc_components_mutex` 的初始值为 0。在 `HWLOC_WIN_SYS` 条件成立时，会使用 `InterlockedCompareExchange()` 函数实现一个基本互斥锁。这个互斥锁的作用是确保在 hwloc_plugins 加载和修改过程中，只有一个进程在执行相关的操作，从而避免多个进程同时注册或卸载组件等行为导致的问题。


```cpp
/* hwloc_components_mutex serializes:
 * - loading/unloading plugins, and modifications of the hwloc_plugins list
 * - calls to ltdl, including in hwloc_check_plugin_namespace()
 * - registration of components with hwloc_disc_component_register()
 *   and hwloc_xml_callbacks_register()
 */
#ifdef HWLOC_WIN_SYS
/* Basic mutex on top of InterlockedCompareExchange() on windows,
 * Far from perfect, but easy to maintain, and way enough given that this code will never be needed for real. */
#include <windows.h>
static LONG hwloc_components_mutex = 0;
#define HWLOC_COMPONENTS_LOCK() do {						\
  while (InterlockedCompareExchange(&hwloc_components_mutex, 1, 0) != 0)	\
    SwitchToThread();								\
} while (0)
```

这段代码定义了一个名为"HWLOC_COMPONENTS_UNLOCK"的宏，其作用是使硬件组件锁定 mutex 并且暂时解除了组件的互斥锁。具体实现如下：

1. 在定义宏之前，先检查当前是否有互斥锁，如果没有，则直接返回。
2. 如果当前存在互斥锁，则使用 pthread_mutex_lock() 函数申请互斥锁，并将其存储在 hwloc_components_mutex 中。
3. 使用 pthread_mutex_lock() 函数获取互斥锁后，如果当前组件已经完成初始化，则执行解锁操作，否则一直等待组件完成初始化。

如果没有互斥锁，则直接使用 pthread_mutex_init() 函数初始化互斥锁，并将其存储在 hwloc_components_mutex 中。

由于在定义宏时使用了 `#elif defined HWLOC_HAVE_PTHREAD_MUTEX` 注释，因此在 HWLOC_HAVE_PTHREAD_MUTEX 条件下，该宏将使用 pthread 库的互斥锁实现。而在 HWLOC_WIN_SYS 条件下，该宏将使用 Windows 系统的互斥锁实现。

另外，该代码中还包含一个定义宏 `HWLOC_COMPONENTS_LOCK`，该宏的作用与 `HWLOC_COMPONENTS_UNLOCK` 相反，即使用 pthread 库的互斥锁实现，并在定义时需要与 `HWLOC_COMPONENTS_UNLOCK` 宏一起使用。


```cpp
#define HWLOC_COMPONENTS_UNLOCK() do {						\
  assert(hwloc_components_mutex == 1);						\
  hwloc_components_mutex = 0;							\
} while (0)

#elif defined HWLOC_HAVE_PTHREAD_MUTEX
/* pthread mutex if available (except on windows) */
#include <pthread.h>
static pthread_mutex_t hwloc_components_mutex = PTHREAD_MUTEX_INITIALIZER;
#define HWLOC_COMPONENTS_LOCK() pthread_mutex_lock(&hwloc_components_mutex)
#define HWLOC_COMPONENTS_UNLOCK() pthread_mutex_unlock(&hwloc_components_mutex)

#else /* HWLOC_WIN_SYS || HWLOC_HAVE_PTHREAD_MUTEX */
#error No mutex implementation available
#endif


```

这段代码是一个C语言的预处理指令，它检查两个条件是否都为真：HWLOC_HAVE_PLUGINS 和 HWLOC_HAVE_LTDL。如果是，那么它将包含以下C代码：
```cppc
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
```
如果没有HWLOC_HAVE_PLUGINS或HWLOC_HAVE_LTDL，那么它将包含以下C代码：
```cppc
#include <linux/version.h>
```
这个预处理指令的作用是检查操作系统是否支持HWLOC_HAVE_PLUGINS和HWLOC_HAVE_LTDL，如果是，那么它将加载一个名为"ltdl-based plugin load"的插件。如果操作系统不支持插件，那么它将包含一个名为"linux-version.h"的预处理指令，用于获取Linux版本号。


```cpp
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
```

这段代码定义了一系列函数，用于支持 .so 文件类型的动态链接库的加载和卸载操作。

具体来说，代码中定义了两个名为 hwloc_dlopen 和 hwloc_dlclose 的函数，用于加载和关闭 .so 文件。这两个函数分别使用 dlopen 和 dlclose 函数实现。此外，定义了 hwloc_dlerror 和 hwloc_dlsym 函数，用于在 .so 文件加载或卸载时出现错误或符号名称不匹配的情况。

接着，定义了 hwloc_dlinit 和 hwloc_dlexit 函数，用于在 .so 文件初始化和卸载时执行一些操作。

最后，定义了两个宏定义，用于将函数名替换为 .so 文件的扩展名。

该代码是一个用于支持 .so 文件类型的动态链接库的加载和卸载操作的库，可以被嵌入到其他应用程序中，以便在需要时动态加载或卸载特定的 .so 文件。


```cpp
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
  (void) asprintf(&filename, "%s.so", _filename);
  if (!filename)
    return NULL;
  handle = dlopen(filename, RTLD_NOW|RTLD_LOCAL);
  free(filename);
  return handle;
}

```



This function appears to be part of the "hwloc" package, which is a software package that aims to provide a unified interface for managing low-level network devices like device drivers and system files. 

The function takes a single parameter of type `void *` and is used to "look up" the path to a top-level directory (i.e., the directory that contains the data files of the lower-level devices) and returns the path, if found. The function uses a combination of absolute paths and relative paths to avoid the possibility of looking into the same directory multiple times.

The function starts by creating a pointer to a memory-efficient buffer for storing the future directory paths, which is initialized with the value of `_paths` from the `hwloc` command-line tool. This buffer is then modified to store the absolute path of the top-level directory that contains the data files of the lower-level devices.

The function then enters a loop that looks for the top-level directory by iterating through the paths stored in the buffer. For each path, it performs the following steps:

1. Look for a directory starting with the specified prefix (e.g., "device0").
2. If the directory is found, look for the next directory node (e.g., a "device0" directory) until it finds the end of the path.
3. If the directory is not found, exit the loop and return an error code.
4. If the directory is found, the function performs the requested function call on the data file specified by the `data` parameter, and then frees the memory allocated for the data file.
5. After the loop finishes, the function frees the memory allocated for the buffer and returns 0 to indicate success.

Note that the function assumes that the path to the top-level directory containing the data files is known at compile-time and that the requested function call is a valid function in the "hwloc" package.


```cpp
static int
hwloc_dlforeachfile(const char *_paths,
		    int (*func)(const char *filename, void *data),
		    void *data)
{
  char *paths = NULL, *path;

  paths = strdup(_paths);
  if (!paths)
    return -1;

  path = paths;
  while (*path) {
    char *colon;
    DIR *dir;
    struct dirent *dirent;

    colon = strchr(path, ':');
    if (colon)
      *colon = '\0';

    if (hwloc_plugins_verbose)
      fprintf(stderr, "hwloc:  Looking under %s\n", path);

    dir = opendir(path);
    if (!dir)
      goto next;

    while ((dirent = readdir(dir)) != NULL) {
      char *abs_name, *suffix;
      struct stat stbuf;
      int err;

      err = asprintf(&abs_name, "%s/%s", path, dirent->d_name);
      if (err < 0)
	continue;

      err = stat(abs_name, &stbuf);
      if (err < 0) {
	free(abs_name);
        continue;
      }
      if (!S_ISREG(stbuf.st_mode)) {
	free(abs_name);
	continue;
      }

      /* Only keep .so files, and remove that suffix to get the component basename */
      suffix = strrchr(abs_name, '.');
      if (!suffix || strcmp(suffix, ".so")) {
	free(abs_name);
	continue;
      }
      *suffix = '\0';

      err = func(abs_name, data);
      if (err) {
	free(abs_name);
	continue;
      }

      free(abs_name);
    }

    closedir(dir);

  next:
    if (!colon)
      break;
    path = colon+1;
  }

  free(paths);
  return 0;
}
```

This is a plugin code that checks the input file (basename) for a valid plugin type, and if it's not a valid type, it prints a message and returns.

The plugin code first checks if the input file has the name "hwloc" followed by a space and a digit, which is the signature of the HWLOC plugin. If it doesn't match, it prints a message and returns.

If the input file has the name "hwloc_xml_" followed by a space, the plugin code checks if the file is an XML description of a hardware location component. If it is, the plugin code checks if the "hwloc" plugin type is set to "DISCOVERY" and prints a message if it's not set to the expected type.

If the input file has a name that doesn't match any of the expected plugins, the plugin code prints a message and returns.

Finally, the plugin code allocates memory for the plugin descriptor and queues it in the list of plugins. If it encounters an error, it prints a message and returns.


```cpp
#endif /* !HWLOC_HAVE_LTDL */

/* array of pointers to dynamically loaded plugins */
static struct hwloc__plugin_desc {
  char *name;
  struct hwloc_component *component;
  char *filename;
  hwloc_dlhandle handle;
  struct hwloc__plugin_desc *next;
} *hwloc_plugins = NULL;

static int
hwloc__dlforeach_cb(const char *filename, void *_data __hwloc_attribute_unused)
{
  const char *basename;
  hwloc_dlhandle handle;
  struct hwloc_component *component;
  struct hwloc__plugin_desc *desc, **prevdesc;
  char *componentsymbolname;

  if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Plugin dlforeach found `%s'\n", filename);

  basename = strrchr(filename, '/');
  if (!basename)
    basename = filename;
  else
    basename++;

  if (hwloc_plugins_blacklist && strstr(hwloc_plugins_blacklist, basename)) {
    if (hwloc_plugins_verbose)
      fprintf(stderr, "hwloc: Plugin `%s' is blacklisted in the environment\n", basename);
    goto out;
  }

  /* dlopen and get the component structure */
  handle = hwloc_dlopenext(filename);
  if (!handle) {
    if (hwloc_plugins_verbose)
      fprintf(stderr, "hwloc: Failed to load plugin: %s\n", hwloc_dlerror());
    goto out;
  }

  componentsymbolname = malloc(strlen(basename)+10+1);
  if (!componentsymbolname) {
    if (hwloc_plugins_verbose)
      fprintf(stderr, "hwloc: Failed to allocation component `%s' symbol\n",
	      basename);
    goto out_with_handle;
  }
  sprintf(componentsymbolname, "%s_component", basename);
  component = hwloc_dlsym(handle, componentsymbolname);
  if (!component) {
    if (hwloc_plugins_verbose)
      fprintf(stderr, "hwloc: Failed to find component symbol `%s'\n",
	      componentsymbolname);
    free(componentsymbolname);
    goto out_with_handle;
  }
  if (component->abi != HWLOC_COMPONENT_ABI) {
    if (hwloc_plugins_verbose)
      fprintf(stderr, "hwloc: Plugin symbol ABI %u instead of %d\n",
	      component->abi, HWLOC_COMPONENT_ABI);
    free(componentsymbolname);
    goto out_with_handle;
  }
  if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Plugin contains expected symbol `%s'\n",
	    componentsymbolname);
  free(componentsymbolname);

  if (HWLOC_COMPONENT_TYPE_DISC == component->type) {
    if (strncmp(basename, "hwloc_", 6)) {
      if (hwloc_plugins_verbose)
	fprintf(stderr, "hwloc: Plugin name `%s' doesn't match its type DISCOVERY\n", basename);
      goto out_with_handle;
    }
  } else if (HWLOC_COMPONENT_TYPE_XML == component->type) {
    if (strncmp(basename, "hwloc_xml_", 10)) {
      if (hwloc_plugins_verbose)
	fprintf(stderr, "hwloc: Plugin name `%s' doesn't match its type XML\n", basename);
      goto out_with_handle;
    }
  } else {
    if (hwloc_plugins_verbose)
      fprintf(stderr, "hwloc: Plugin name `%s' has invalid type %u\n",
	      basename, (unsigned) component->type);
    goto out_with_handle;
  }

  /* allocate a plugin_desc and queue it */
  desc = malloc(sizeof(*desc));
  if (!desc)
    goto out_with_handle;
  desc->name = strdup(basename);
  desc->filename = strdup(filename);
  desc->component = component;
  desc->handle = handle;
  desc->next = NULL;
  if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Plugin descriptor `%s' ready\n", basename);

  /* append to the list */
  prevdesc = &hwloc_plugins;
  while (*prevdesc)
    prevdesc = &((*prevdesc)->next);
  *prevdesc = desc;
  if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Plugin descriptor `%s' queued\n", basename);
  return 0;

 out_with_handle:
  hwloc_dlclose(handle);
 out:
  return 0;
}

```

这段代码是一个C语言的函数，定义了一个名为"hwloc_plugins_exit()"的静态函数。

这个函数的作用是当插件程序运行结束时，对其进行清理操作，包括关闭所有打开的插件，释放相关资源，最后将插件链表指向 NULL，使得以后再次打开插件时不再读取过来。

具体实现中，首先定义了一个结构体"hwloc__plugin_desc"，用于存储每个插件的描述信息，包括插件名称、文件名、文件路径、打开时间等信息。

接着，函数内部使用嵌套循环，遍历每个插件，对其进行以下操作：

1. 关闭当前插件对应文件描述符
2. 释放当前插件的名称、文件名、文件路径等信息，将当前插件指针指向下一个插件
3. 将当前插件指针指向下一个插件
4. 循环结束后，将"hwloc_plugins"变量设置为NULL，表示插件链表已经空，不再接收新的插件
5. 调用"hwloc_dlexit()"函数，使得程序退出

这个函数是插件程序结束时必须要执行的一个函数，可以确保在插件程序结束时，对其进行清理，使得下一次启动插件时不会出现乱序加载的情况。


```cpp
static void
hwloc_plugins_exit(void)
{
  struct hwloc__plugin_desc *desc, *next;

  if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Closing all plugins\n");

  desc = hwloc_plugins;
  while (desc) {
    next = desc->next;
    hwloc_dlclose(desc->handle);
    free(desc->name);
    free(desc->filename);
    free(desc);
    desc = next;
  }
  hwloc_plugins = NULL;

  hwloc_dlexit();
}

```

该代码是一个C语言函数，名为`hwloc_plugins_init()`，用于初始化HWLOC插件。

该函数首先定义了两个静态变量`verboseenv`和`hwloc_plugins_blacklist`，分别用于获取/设置HWLOC插件的verbose模式和黑名单。

接下来，函数使用`getenv()`函数获取`HWLOC_PLUGINS_VERBOSE`和`HWLOC_PLUGINS_BLACKLIST`环境变量，其中`getenv()`函数会尝试从当前进程的环境中查找相应的环境变量，如果找不到，则返回一个默认值0。

接着，函数使用`atoi()`函数从`verboseenv`中获取一个字符串，并将其转换为整数类型。

然后，函数调用`hwloc_dlinit()`函数，并检查其是否出错。如果出错，函数跳转到`out`标签，否则继续执行后续代码。

接下来，函数使用`getenv()`函数获取`HWLOC_PLUGINS_PATH`环境变量，如果该环境变量存在，则将其设置为函数的参数。

然后，函数创建一个名为`hwloc_plugins`的空指针，用于存储初始化的插件。

接着，函数使用`fprintf()`函数在调试输出中输出函数的路径。然后，函数调用`hwloc_dlforeachfile()`函数，该函数用于遍历指定路径下的文件，并将下载的插件载入到`hwloc_plugins`指向的内存区域中。如果函数执行成功，则跳转到`out_with_init()`标签，否则继续执行后续代码。

最后，函数返回0表示初始化成功，如果初始化失败，则返回-1。


```cpp
static int
hwloc_plugins_init(void)
{
  const char *verboseenv;
  const char *path = HWLOC_PLUGINS_PATH;
  const char *env;
  int err;

  verboseenv = getenv("HWLOC_PLUGINS_VERBOSE");
  hwloc_plugins_verbose = verboseenv ? atoi(verboseenv) : 0;

  hwloc_plugins_blacklist = getenv("HWLOC_PLUGINS_BLACKLIST");

  err = hwloc_dlinit();
  if (err)
    goto out;

  env = getenv("HWLOC_PLUGINS_PATH");
  if (env)
    path = env;

  hwloc_plugins = NULL;

  if (hwloc_plugins_verbose)
    fprintf(stderr, "hwloc: Starting plugin dlforeach in %s\n", path);
  err = hwloc_dlforeachfile(path, hwloc__dlforeach_cb, NULL);
  if (err)
    goto out_with_init;

  return 0;

 out_with_init:
  hwloc_plugins_exit();
 out:
  return -1;
}

```

It looks like there is a function called `register_discovery_component` that is used to register a new discovery component with the `hwloc` library. This function takes a component struct as an argument and returns either 0 on success or -1 on failure.

The function starts by initializing a pointer to the previous component in the linked list, and then loops through the list until it finds a component with the same name as the current component. If it finds such a component, it compares the priorities of the existing component to the new component and drops the new component if the new component has a higher priority.

If the loop completes without finding a matching component, the function returns -1. It is also possible to register multiple components in a single call to the `register_discovery_component` function by passing multiple `component` structs as an argument.

There is also a function called `get_discovery_component_by_name` that takes a name as an argument and returns the first component in the list that has that name. This function is not used in the `register_discovery_component` function, but it is included here for reference.


```cpp
#endif /* HWLOC_HAVE_PLUGINS */

static int
hwloc_disc_component_register(struct hwloc_disc_component *component,
			      const char *filename)
{
  struct hwloc_disc_component **prev;

  /* check that the component name is valid */
  if (!strcmp(component->name, HWLOC_COMPONENT_STOP_NAME)) {
    if (hwloc_components_verbose)
      fprintf(stderr, "hwloc: Cannot register discovery component with reserved name `" HWLOC_COMPONENT_STOP_NAME "'\n");
    return -1;
  }
  if (strchr(component->name, HWLOC_COMPONENT_EXCLUDE_CHAR)
      || strchr(component->name, HWLOC_COMPONENT_PHASESEP_CHAR)
      || strcspn(component->name, HWLOC_COMPONENT_SEPS) != strlen(component->name)) {
    if (hwloc_components_verbose)
      fprintf(stderr, "hwloc: Cannot register discovery component with name `%s' containing reserved characters `%c" HWLOC_COMPONENT_SEPS "'\n",
	      component->name, HWLOC_COMPONENT_EXCLUDE_CHAR);
    return -1;
  }

  /* check that the component phases are valid */
  if (!component->phases
      || (component->phases != HWLOC_DISC_PHASE_GLOBAL
	  && component->phases & ~(HWLOC_DISC_PHASE_CPU
				   |HWLOC_DISC_PHASE_MEMORY
				   |HWLOC_DISC_PHASE_PCI
				   |HWLOC_DISC_PHASE_IO
				   |HWLOC_DISC_PHASE_MISC
				   |HWLOC_DISC_PHASE_ANNOTATE
				   |HWLOC_DISC_PHASE_TWEAK))) {
    if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Cannot register discovery component `%s' with invalid phases 0x%x\n",
              component->name, component->phases);
    return -1;
  }

  prev = &hwloc_disc_components;
  while (NULL != *prev) {
    if (!strcmp((*prev)->name, component->name)) {
      /* if two components have the same name, only keep the highest priority one */
      if ((*prev)->priority < component->priority) {
	/* drop the existing component */
	if (hwloc_components_verbose)
	  fprintf(stderr, "hwloc: Dropping previously registered discovery component `%s', priority %u lower than new one %u\n",
		  (*prev)->name, (*prev)->priority, component->priority);
	*prev = (*prev)->next;
      } else {
	/* drop the new one */
	if (hwloc_components_verbose)
	  fprintf(stderr, "hwloc: Ignoring new discovery component `%s', priority %u lower than previously registered one %u\n",
		  component->name, component->priority, (*prev)->priority);
	return -1;
      }
    }
    prev = &((*prev)->next);
  }
  if (hwloc_components_verbose)
    fprintf(stderr, "hwloc: Registered discovery component `%s' phases 0x%x with priority %u (%s%s)\n",
	    component->name, component->phases, component->priority,
	    filename ? "from plugin " : "statically build", filename ? filename : "");

  prev = &hwloc_disc_components;
  while (NULL != *prev) {
    if ((*prev)->priority < component->priority)
      break;
    prev = &((*prev)->next);
  }
  component->next = *prev;
  *prev = component;
  return 0;
}

```

这段代码定义了一个名为 "hwloc_component_finalize_cbs" 的函数，以及一个名为 "hwloc_component_finalize_cb_count" 的整数变量。

"hwloc_component_finalize_cbs" 函数是一个静态函数，它在组件被卸载时被调用。它接受一个 "unsigned long" 类型的参数，表示要将被卸载的组件的 ID。

"hwloc_component_finalize_cb_count" 整数变量记录了已注册的 "hwloc_component_finalize_cbs" 函数的个数。

代码中还定义了一个名为 "hwloc_components_init" 的函数。该函数在没有定义任何参数的情况下被定义，但似乎没有实际的作用。

最后，在 "hwloc_component_finalize_cbs" 函数内部，使用了一些环境变量，包括 "HWLOC_COMPONENTS_VERBOSE" 变量，似乎用来设置组件的verbose输出。


```cpp
#include "static-components.h"

static void (**hwloc_component_finalize_cbs)(unsigned long);
static unsigned hwloc_component_finalize_cb_count;

void
hwloc_components_init(void)
{
#ifdef HWLOC_HAVE_PLUGINS
  struct hwloc__plugin_desc *desc;
#endif
  const char *verboseenv;
  unsigned i;

  HWLOC_COMPONENTS_LOCK();
  assert((unsigned) -1 != hwloc_components_users);
  if (0 != hwloc_components_users++) {
    HWLOC_COMPONENTS_UNLOCK();
    return;
  }

  verboseenv = getenv("HWLOC_COMPONENTS_VERBOSE");
  hwloc_components_verbose = verboseenv ? atoi(verboseenv) : 0;

```

除了解析静态组件和处理hwloc_static_components数组外，还处理了一个名为"dynamic"的动态组件。在此部分，我假设您已经定义了`hwloc_dynamic_components`，并且遵循了以下逻辑：

1. 如果动态组件存在，则将其显示。
2. 对于每个动态组件，如果设置了`hwloc_dynamic_component_finalize_cb`，则将其设置为`hwloc_dynamic_component_finalize_cb`。
3. 对于每个动态组件，如果`hwloc_dynamic_component_register_cb`被设置为`NULL`，则将其添加到`hwloc_dynamic_components_queue`中。
4. 对于每个动态组件，如果`hwloc_dynamic_component_finalize_cb`被设置为`NULL`，则将其添加到`hwloc_dynamic_components_queue`中。
5. 在动态组件卸载处理程序中，如果`hwloc_dynamic_component_finalize_cb`被设置为`NULL`，则将其从`hwloc_dynamic_components_queue`中删除。
6. 在动态组件卸载处理程序中，如果`hwloc_dynamic_component_register_cb`被设置为`NULL`，则不会将其添加到`hwloc_dynamic_components_queue`中。

请注意，我假设您已经实现了`hwloc_dynamic_component_finalize_cb`和`hwloc_dynamic_component_register_cb`函数。如果尚未实现，请按照以上逻辑进行实现。


```cpp
#ifdef HWLOC_HAVE_PLUGINS
  hwloc_plugins_init();
#endif

  hwloc_component_finalize_cbs = NULL;
  hwloc_component_finalize_cb_count = 0;
  /* count the max number of finalize callbacks */
  for(i=0; NULL != hwloc_static_components[i]; i++)
    hwloc_component_finalize_cb_count++;
#ifdef HWLOC_HAVE_PLUGINS
  for(desc = hwloc_plugins; NULL != desc; desc = desc->next)
    hwloc_component_finalize_cb_count++;
#endif
  if (hwloc_component_finalize_cb_count) {
    hwloc_component_finalize_cbs = calloc(hwloc_component_finalize_cb_count,
					  sizeof(*hwloc_component_finalize_cbs));
    assert(hwloc_component_finalize_cbs);
    /* forget that max number and recompute the real one below */
    hwloc_component_finalize_cb_count = 0;
  }

  /* hwloc_static_components is created by configure in static-components.h */
  for(i=0; NULL != hwloc_static_components[i]; i++) {
    if (hwloc_static_components[i]->flags) {
      if (HWLOC_SHOW_CRITICAL_ERRORS())
        fprintf(stderr, "hwloc: Ignoring static component with invalid flags %lx\n",
                hwloc_static_components[i]->flags);
      continue;
    }

    /* initialize the component */
    if (hwloc_static_components[i]->init && hwloc_static_components[i]->init(0) < 0) {
      if (hwloc_components_verbose)
	fprintf(stderr, "hwloc: Ignoring static component, failed to initialize\n");
      continue;
    }
    /* queue ->finalize() callback if any */
    if (hwloc_static_components[i]->finalize)
      hwloc_component_finalize_cbs[hwloc_component_finalize_cb_count++] = hwloc_static_components[i]->finalize;

    /* register for real now */
    if (HWLOC_COMPONENT_TYPE_DISC == hwloc_static_components[i]->type)
      hwloc_disc_component_register(hwloc_static_components[i]->data, NULL);
    else if (HWLOC_COMPONENT_TYPE_XML == hwloc_static_components[i]->type)
      hwloc_xml_callbacks_register(hwloc_static_components[i]->data);
    else
      assert(0);
  }

  /* dynamic plugins */
```

这段代码是一个用于检查HWLOC_HAVE_PLUGINS环境元素的if语句。如果环境中存在插件，则按照插件的名称逐个处理组件。如果组件的属性值中包含`__attribute__(( plasma ))`或`__attribute__((__dep__))`，则会输出一条信息，否则跳过该组件。

具体来说，代码的作用是遍历插件中的所有组件，对于每个组件，先检查其属性中是否包含`__attribute__(( plasma ))`或`__attribute__((__dep__))`，如果是则输出一条信息，否则执行组件的初始化或最终化操作，并注册一个`finalize` callback给组件。对于xml类型的组件，会注册一个`finalize`的xml回调。

注册完最终化 callback之后，将输出一条信息，表明所有组件都已经被正确初始化或注册。


```cpp
#ifdef HWLOC_HAVE_PLUGINS
  for(desc = hwloc_plugins; NULL != desc; desc = desc->next) {
    if (desc->component->flags) {
      if (HWLOC_SHOW_CRITICAL_ERRORS())
        fprintf(stderr, "hwloc: Ignoring plugin `%s' component with invalid flags %lx\n",
                desc->name, desc->component->flags);
      continue;
    }

    /* initialize the component */
    if (desc->component->init && desc->component->init(0) < 0) {
      if (hwloc_components_verbose)
	fprintf(stderr, "hwloc: Ignoring plugin `%s', failed to initialize\n", desc->name);
      continue;
    }
    /* queue ->finalize() callback if any */
    if (desc->component->finalize)
      hwloc_component_finalize_cbs[hwloc_component_finalize_cb_count++] = desc->component->finalize;

    /* register for real now */
    if (HWLOC_COMPONENT_TYPE_DISC == desc->component->type)
      hwloc_disc_component_register(desc->component->data, desc->filename);
    else if (HWLOC_COMPONENT_TYPE_XML == desc->component->type)
      hwloc_xml_callbacks_register(desc->component->data);
    else
      assert(0);
  }
```

这段代码是针对hwloc_topology_components_init函数的定义，其作用是初始化hwloc_topology结构体中的nr_blacklisted_components、blacklisted_components、backends、backend_phases和backend_excluded_phases成员。

具体来说，代码首先执行topology->nr_blacklisted_components = 0；这个语句将初始化nr_blacklisted_components成员，使其初始值为0。接着，代码定义了两个名为topology->blacklisted_components和topology->backends的变量，并将它们初始化为 NULL，即没有内存分配。

随后，代码定义了四个名为topology->backend_phases、topology->backend_excluded_phases和topology->backends_layers的成员，并将它们初始化为0，即没有内存分配。最后，代码没有做任何其他事情，直接返回了。


```cpp
#endif

  HWLOC_COMPONENTS_UNLOCK();
}

void
hwloc_topology_components_init(struct hwloc_topology *topology)
{
  topology->nr_blacklisted_components = 0;
  topology->blacklisted_components = NULL;

  topology->backends = NULL;
  topology->backend_phases = 0;
  topology->backend_excluded_phases = 0;
}

```

这段代码定义了一个名为 `hwloc_disc_component_find` 的函数，用于查找给定名称的硬件设备组件。函数接受两个参数：要查找的名称和工作区结束符（`endp` 参数），返回满足条件的硬件设备组件指针（`comp` 参数）。

函数首先定义了一个名为 `comp` 的结构体变量，用于存储要查找的组件。然后定义了一个名为 `end` 的指针变量，用于存储找到的组件名称的结束符。接着，定义了一个名为 `strchr` 的函数，用于在给定的名称中查找 `HWLOC_COMPONENT_PHASESEP_CHAR` 字符（即组件名称中的偏移符）。

函数的第一个实现部分，定义了一个从 `hwloc_disc_components` 开始，以 `NULL` 为结束符的循环，用于存储组件列表。在每次循环中，首先将当前组件的名称与给定的名称进行比较，如果它们相等，就返回当前组件（即返回 `comp`）。否则，递归地处理组件列表中的下一个组件。

函数的第二个实现部分，将 `endp` 指向组件名称的结束符，使得可以在组件列表中继续查找。

总的来说，这段代码定义了一个用于查找给定名称的硬件设备组件的函数，可以用于许多不同的硬件设备，如磁盘驱动器、打印机墨盒等。


```cpp
/* look for name among components, ignoring things after `:' */
static struct hwloc_disc_component *
hwloc_disc_component_find(const char *name, const char **endp)
{
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

```

这段代码定义了一个名为 `hwloc_phases_from_string` 的函数，它接受一个字符串参数 `s`。该函数的作用是尝试将 `s` 中的字符映射为相应的奇偶阶段，并返回该奇偶阶段。

函数的实现分为以下几个步骤：

1. 如果 `s` 为空，函数返回 0，表示没有找到任何有效的奇偶阶段。
2. 如果 `s` 中的第一个字符不是 '0' 或 '9'，函数先尝试将 `s` 转换为 `"global"`，如果是，函数返回 `HWLOC_DISC_PHASE_GLOBAL`；如果不是，函数尝试将 `s` 转换为 `"cpu"`，如果是，函数返回 `HWLOC_DISC_PHASE_CPU`；以此类推，如果 `s` 不能转换为任何一个指定的奇偶阶段，函数返回 0。
3. 如果 `s` 中的第一个字符是 '0' 或 '9'，函数将尝试使用 `strtoul` 函数将 `s` 的本义字符转换为整数，并返回该整数。
4. 如果 `s` 中既没有第一个字符为 '0' 或 '9'，也没有本义字符可以转换为整数，函数将返回 0。

该函数可以作为一个用于 HWLOC-DISC_PHASE_ 奇偶阶段输出的函数，根据输入的字符串选择相应的奇偶阶段，并返回对应的奇偶阶段编号。


```cpp
static unsigned
hwloc_phases_from_string(const char *s)
{
  if (!s)
    return ~0U;
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
  return (unsigned) strtoul(s, NULL, 0);
}

```

Replace `linuxpci` and `linuxio` with `linux` (with IO phases)
================================================================

As mentioned in the original post, we need to replace the old `linuxpci` and `linuxio` with the new `linux` (with IO phases) in order to have backward compatibility with the pre-v2.0 and v2.0 versions respectively.

If `hwloc_components_verbose` is set to `1`, we will print a message to the console. We will first look for an existing component with the name `linux` and then determine the IO phases. If the `linux` component is found, we will update its IO phases to the new `linux` phase.

If the `hwloc_disc_component_find` function returns nothing, we will assume that the Linux driver is not installed and try to blacklist the component with the deprecated name. We will use the `hwloc_phases_from_string` function to get the current IO phases of the `linux` component, and then we will add the new `linux` phase to it.

If the `hwloc_components_verbose` is set to `0`, we will use the `hwloc_disc_component_find` function to get the current IO phases of the `linux` component.

If the `linux` component is found, we will update its IO phases to the new `linux` phase.

If the `hwloc_disc_component_find` function returns nothing, we will assume that the Linux driver is not installed and try to blacklist the component with the deprecated name.

We will then return the result of the replacement.


```cpp
static int
hwloc_disc_component_blacklist_one(struct hwloc_topology *topology,
				   const char *name)
{
  struct hwloc_topology_forced_component_s *blacklisted;
  struct hwloc_disc_component *comp;
  unsigned phases;
  unsigned i;

  if (!strcmp(name, "linuxpci") || !strcmp(name, "linuxio")) {
    /* replace linuxpci and linuxio with linux (with IO phases)
     * for backward compatibility with pre-v2.0 and v2.0 respectively */
    if (hwloc_components_verbose)
      fprintf(stderr, "hwloc: Replacing deprecated component `%s' with `linux' IO phases in blacklisting\n", name);
    comp = hwloc_disc_component_find("linux", NULL);
    phases = HWLOC_DISC_PHASE_PCI | HWLOC_DISC_PHASE_IO | HWLOC_DISC_PHASE_MISC | HWLOC_DISC_PHASE_ANNOTATE;

  } else {
    /* normal lookup */
    const char *end;
    comp = hwloc_disc_component_find(name, &end);
    phases = hwloc_phases_from_string(end);
  }
  if (!comp) {
    errno = EINVAL;
    return -1;
  }

  if (hwloc_components_verbose)
    fprintf(stderr, "hwloc: Blacklisting component `%s` phases 0x%x\n", comp->name, phases);

  for(i=0; i<topology->nr_blacklisted_components; i++) {
    if (topology->blacklisted_components[i].component == comp) {
      topology->blacklisted_components[i].phases |= phases;
      return 0;
    }
  }

  blacklisted = realloc(topology->blacklisted_components, (topology->nr_blacklisted_components+1)*sizeof(*blacklisted));
  if (!blacklisted)
    return -1;

  blacklisted[topology->nr_blacklisted_components].component = comp;
  blacklisted[topology->nr_blacklisted_components].phases = phases;
  topology->blacklisted_components = blacklisted;
  topology->nr_blacklisted_components++;
  return 0;
}

```

这段代码是一个名为 `hwloc_topology_set_components` 的函数，它是 `hwloc_topology_compute_topology` 的安全子函数。用于设置硬件抽象层（HAL）中的组件。

具体来说，这段代码的作用是：

1. 如果 `topology` 已经加载完成，检查是否有冲突，如果有冲突，则返回错误。
2. 如果设置的组件中不包含名为 "all" 的组件，或者组件包含名为 "all" 的组件但后缀不是 "hwloc_component_phasesep_char"，则返回错误。
3. 如果设置的组件是黑色的，尝试从 `name` 参数中提取出 "all" 字符串中的后缀 "hwloc_component_phasesep_char"，然后将其赋值给 `topology->backend_excluded_phases`。
4. 如果设置的组件中包含名为 "all" 的组件，尝试将其值作为 `hwloc_phases_from_string` 函数的第一个参数，如果该函数返回的值可以作为 `topology->backend_excluded_phases` 的值，则返回 0。

如果出现任何错误，函数将返回 -1，否则返回 0。


```cpp
int
hwloc_topology_set_components(struct hwloc_topology *topology,
			      unsigned long flags,
			      const char *name)
{
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  if (flags & ~HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST) {
    errno = EINVAL;
    return -1;
  }

  /* this flag is strictly required for now */
  if (flags != HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST) {
    errno = EINVAL;
    return -1;
  }

  if (!strncmp(name, "all", 3) && name[3] == HWLOC_COMPONENT_PHASESEP_CHAR) {
    topology->backend_excluded_phases = hwloc_phases_from_string(name+4);
    return 0;
  }

  return hwloc_disc_component_blacklist_one(topology, name);
}

```

这段代码定义了一个名为 `hwloc_disc_component_force_enable` 的函数，它属于 `hwloc_disc_component_find` 的函数指针。这个函数的作用是，如果两个组件都加载了，就从第一个组件中实例化，并将指定的环境变量 `envvar_forced` 设置为 `true`，然后尝试启动它。如果第一个组件实例化失败，就返回 `-1`，否则继续执行，如果启动成功，就返回 `0`，否则返回 `-1`。

这段代码的具体实现可能会在不同的 `hwloc_disc_component` 驱动程序中有所不同，但是它的基本功能是相同的：通过 `hwloc_disc_component_find` 函数查找指定的组件，然后使用 `comp->instantiate` 函数启动它，并设置指定的环境变量。


```cpp
/* used by set_xml(), set_synthetic(), ... environment variables, ... to force the first backend */
int
hwloc_disc_component_force_enable(struct hwloc_topology *topology,
				  int envvar_forced,
				  const char *name,
				  const void *data1, const void *data2, const void *data3)
{
  struct hwloc_disc_component *comp;
  struct hwloc_backend *backend;

  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  comp = hwloc_disc_component_find(name, NULL);
  if (!comp) {
    errno = ENOSYS;
    return -1;
  }

  backend = comp->instantiate(topology, comp, 0U /* force-enabled don't get any phase blacklisting */,
			      data1, data2, data3);
  if (backend) {
    int err;
    backend->envvar_forced = envvar_forced;
    if (topology->backends)
      hwloc_backends_disable_all(topology);
    err = hwloc_backend_enable(backend);

    if (comp->phases == HWLOC_DISC_PHASE_GLOBAL) {
      char *env = getenv("HWLOC_ANNOTATE_GLOBAL_COMPONENTS");
      if (env && atoi(env))
	topology->backend_excluded_phases &= ~HWLOC_DISC_PHASE_ANNOTATE;
    }

    return err;
  } else
    return -1;
}

```

这段代码是一个名为 `hwloc_disc_component_try_enable` 的函数，属于 `hwloc_topology` 和 `hwloc_disc_component` 结构体的成员。

它的作用是检查在给定的拓扑中，指定的组件是否已经被创建过，如果已经创建过，就允许组件创建；如果还没有创建过，则创建组件，设置相应的 phase 和 enable 标志，最后返回是否成功创建。

具体实现包括以下步骤：

1. 获取要操作的 backend。
2. 如果要禁止的 phases 列表与指定的 backend 中的 phases 列表不包括当前要禁止的 phases，则允许组件创建并返回成功。
3. 如果要禁止的 phases 列表包括当前要禁止的 phases，则不允许组件创建，并输出相应的错误信息。
4. 调用 `hwloc_backend_enable` 函数，以允许组件创建。

这个函数的作用是确保在指定的拓扑中，指定的组件可以成功创建，如果不能成功创建，则输出相应的错误信息。


```cpp
static int
hwloc_disc_component_try_enable(struct hwloc_topology *topology,
				struct hwloc_disc_component *comp,
				int envvar_forced,
				unsigned blacklisted_phases)
{
  struct hwloc_backend *backend;

  if (!(comp->phases & ~(topology->backend_excluded_phases | blacklisted_phases))) {
    /* all this backend phases are already excluded, exclude the backend entirely */
    if (hwloc_components_verbose)
      /* do not warn if envvar_forced since system-wide HWLOC_COMPONENTS must be silently ignored after set_xml() etc.
       */
      fprintf(stderr, "hwloc: Excluding discovery component `%s' phases 0x%x, conflicts with excludes 0x%x\n",
	      comp->name, comp->phases, topology->backend_excluded_phases);
    return -1;
  }

  backend = comp->instantiate(topology, comp, topology->backend_excluded_phases | blacklisted_phases,
			      NULL, NULL, NULL);
  if (!backend) {
    if (hwloc_components_verbose || (envvar_forced && HWLOC_SHOW_CRITICAL_ERRORS()))
      fprintf(stderr, "hwloc: Failed to instantiate discovery component `%s'\n", comp->name);
    return -1;
  }

  backend->phases &= ~blacklisted_phases;
  backend->envvar_forced = envvar_forced;
  return hwloc_backend_enable(backend);
}

```

首先，这段代码的作用是遍历一个硬件设备（如声卡）的所有发现组件（disc component），并检查它们是否已被禁用（blacklisted）。如果禁用状态为1（即禁用），则尝试启用（try to enable）。

具体来说，代码分为以下几个部分：

1. 定义变量：定义了一些形参和局部变量。

2. 遍历发现组件：从硬件设备的前端开始，遍历所有的发现组件，直到遍历到最后一个组件。

3. 检查禁用状态：对于每个发现组件，检查它的禁用状态（即 phases 变量是否为 0x00000000）。

4. 尝试启用：如果发现组件的禁用状态为 0，则尝试将其启用（try to enable）。

5. 输出日志：如果启用失败，则会输出一条日志信息。

6. 恢复变量：在遍历过程中，将之前定义的变量（如 curenv 和 blacklisted_phases）保留在栈中，以便在遍历结束后恢复它们。

7. 循环结束后，如果仍然存在禁用组件，则尝试启用它们（hwloc_disc_component_try_enable）。


```cpp
void
hwloc_disc_components_enable_others(struct hwloc_topology *topology)
{
  struct hwloc_disc_component *comp;
  struct hwloc_backend *backend;
  int tryall = 1;
  const char *_env;
  char *env; /* we'll to modify the env value, so duplicate it */
  unsigned i;

  _env = getenv("HWLOC_COMPONENTS");
  env = _env ? strdup(_env) : NULL;

  /* blacklist disabled components */
  if (env) {
    char *curenv = env;
    size_t s;

    while (*curenv) {
      s = strcspn(curenv, HWLOC_COMPONENT_SEPS);
      if (s) {
	char c;

	if (curenv[0] != HWLOC_COMPONENT_EXCLUDE_CHAR)
	  goto nextname;

	/* save the last char and replace with \0 */
	c = curenv[s];
	curenv[s] = '\0';

	/* blacklist it, and just ignore failures to allocate */
	hwloc_disc_component_blacklist_one(topology, curenv+1);

	/* remove that blacklisted name from the string */
	for(i=0; i<s; i++)
	  curenv[i] = *HWLOC_COMPONENT_SEPS;

	/* restore chars (the second loop below needs env to be unmodified) */
	curenv[s] = c;
      }

    nextname:
      curenv += s;
      if (*curenv)
	/* Skip comma */
	curenv++;
    }
  }

  /* enable explicitly listed components */
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

	/* save the last char and replace with \0 */
	c = curenv[s];
	curenv[s] = '\0';

	name = curenv;
	if (!strcmp(name, "linuxpci") || !strcmp(name, "linuxio")) {
	  if (hwloc_components_verbose)
	    fprintf(stderr, "hwloc: Replacing deprecated component `%s' with `linux' in envvar forcing\n", name);
	  name = "linux";
	}

	comp = hwloc_disc_component_find(name, NULL /* we enable the entire component, phases must be blacklisted separately */);
	if (comp) {
	  unsigned blacklisted_phases = 0U;
	  for(i=0; i<topology->nr_blacklisted_components; i++)
	    if (comp == topology->blacklisted_components[i].component) {
	      blacklisted_phases = topology->blacklisted_components[i].phases;
	      break;
	    }
	  if (comp->phases & ~blacklisted_phases)
	    hwloc_disc_component_try_enable(topology, comp, 1 /* envvar forced */, blacklisted_phases);
	} else {
          if (HWLOC_SHOW_CRITICAL_ERRORS())
            fprintf(stderr, "hwloc: Cannot find discovery component `%s'\n", name);
	}

	/* restore chars (the second loop below needs env to be unmodified) */
	curenv[s] = c;
      }

      curenv += s;
      if (*curenv)
	/* Skip comma */
	curenv++;
    }
  }

  /* env is still the same, the above loop didn't modify it */

  /* now enable remaining components (except the explicitly '-'-listed ones) */
  if (tryall) {
    comp = hwloc_disc_components;
    while (NULL != comp) {
      unsigned blacklisted_phases = 0U;
      if (!comp->enabled_by_default)
	goto nextcomp;
      /* check if this component was blacklisted by the application */
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
```

这段代码是一个 C 语言函数，它属于 NH namespaces。函数的作用是打印出系统中所有可用的组件，并输出一个 summary。

具体来说，函数接收一个 NH named table topology 对象，然后遍历所有的组件，将每个组件的名称和状态打印出来。如果设置了 hwloc_components_verbose 环境选项，则会输出更加详细的信息，包括组件的驱动器和内存占用情况。

函数中还有一个 print_enabled_discovery_components 函数，这个函数的作用与上面的函数类似，但是没有将所有组件名称打印出来，只会打印 enabled  discovery components。这个函数是在 print_device_discovery_components 函数中定义的。


```cpp
nextcomp:
      comp = comp->next;
    }
  }

  if (hwloc_components_verbose) {
    /* print a summary */
    int first = 1;
    backend = topology->backends;
    fprintf(stderr, "hwloc: Final list of enabled discovery components: ");
    while (backend != NULL) {
      fprintf(stderr, "%s%s(0x%x)", first ? "" : ",", backend->component->name, backend->phases);
      backend = backend->next;
      first = 0;
    }
    fprintf(stderr, "\n");
  }

  free(env);
}

```

这段代码是一个C语言函数，名为`hwloc_components_fini()`，其作用是当硬件抽象层（HWLOC）组件完成初始化后，释放相关的资源和重置抽象层组件数。

具体来说，代码首先获取当前用户数，如果当前用户数大于0，说明有组件正在被初始化，需要取消初始化并返回。如果当前用户数为0，说明所有组件都已经完成初始化，可以释放资源并执行后续操作。

接下来，代码遍历抽象层组件完成预热的回调函数数，对每个回调函数执行一次。这里的回调函数是在组件完成初始化后执行的，所以可以认为它们是"最终结束"初始化过程的回调函数。

在完成初始化后，代码还需要释放之前分配的内存，将`hwloc_component_finalize_cbs`和`hwloc_component_finalize_cbs`指向内存的起始地址，以确保它们不会被再次使用。

此外，代码还执行了一些其他的操作，如将`hwloc_disc_components`设置为`NULL`，以及重置`hwloc_xml_callbacks_reset()`函数，这里似乎没有具体的解释。


```cpp
void
hwloc_components_fini(void)
{
  unsigned i;

  HWLOC_COMPONENTS_LOCK();
  assert(0 != hwloc_components_users);
  if (0 != --hwloc_components_users) {
    HWLOC_COMPONENTS_UNLOCK();
    return;
  }

  for(i=0; i<hwloc_component_finalize_cb_count; i++)
    hwloc_component_finalize_cbs[hwloc_component_finalize_cb_count-i-1](0);
  free(hwloc_component_finalize_cbs);
  hwloc_component_finalize_cbs = NULL;
  hwloc_component_finalize_cb_count = 0;

  /* no need to unlink/free the list of components, they'll be unloaded below */

  hwloc_disc_components = NULL;
  hwloc_xml_callbacks_reset();

```

这段代码是一个C语言的程序，定义了两个函数，其作用如下：

1. `hwloc_plugins_exit()`函数的作用是输出所有的硬件插件，并关闭插件栈的清理工作。

2. `hwloc_backend_alloc()`函数的作用是分配内存空间，用于一个新的 `hwloc_backend` 结构体，并将其构造成 `hwloc_topology` 和 `struct hwloc_disc_component` 结构体。

`hwloc_plugins_exit()`函数会在系统启动时被调用一次，此时所有的插件，包括桌面应用程序和用户态程序，都将被杀死，并关闭插件栈的清理工作。

`hwloc_backend_alloc()`函数用于在 `hwloc_topology` 和 `struct hwloc_disc_component` 两者之间选择并设置 `hwloc_backend` 结构体的一些成员变量，例如 `component` 和 `topology` 成员变量，以及 `phases` 成员变量的减法运算。

`hwloc_backend_alloc()` 函数的实现比较复杂，它的作用是确保 `hwloc_topology` 和 `struct hwloc_disc_component` 的 `component` 和 `topology` 成员变量被正确初始化，并且在 `hwloc_backend` 结构体中保存下来。

`hwloc_plugins_exit()` 和 `hwloc_backend_alloc()` 函数都在 `hwloc` 库中定义，用于管理操作系统插件栈和硬件组件的发现工作。


```cpp
#ifdef HWLOC_HAVE_PLUGINS
  hwloc_plugins_exit();
#endif

  HWLOC_COMPONENTS_UNLOCK();
}

struct hwloc_backend *
hwloc_backend_alloc(struct hwloc_topology *topology,
		    struct hwloc_disc_component *component)
{
  struct hwloc_backend * backend = malloc(sizeof(*backend));
  if (!backend) {
    errno = ENOMEM;
    return NULL;
  }
  backend->component = component;
  backend->topology = topology;
  /* filter-out component phases that are excluded */
  backend->phases = component->phases & ~topology->backend_excluded_phases;
  if (backend->phases != component->phases && hwloc_components_verbose)
    fprintf(stderr, "hwloc: Trying discovery component `%s' with phases 0x%x instead of 0x%x\n",
	    component->name, backend->phases, component->phases);
  backend->flags = 0;
  backend->discover = NULL;
  backend->get_pci_busid_cpuset = NULL;
  backend->disable = NULL;
  backend->is_thissystem = -1;
  backend->next = NULL;
  backend->envvar_forced = 0;
  return backend;
}

```

This function appears to enable a discovery component for a specific backend. It does this by first checking if the backend already has flags enabled for discovery components, and if so, it enables the component. If the flags are not enabled, it enables the component and returns an error.

If the backend does not already have flags enabled for discovery components, it creates a new linked list of backends and makes the new linked list the next in the list. It then enqueues the new backend at the end of the linked list.

Finally, it updates the topology object to reflect the new state of the backends. This includes updating the backend's phases and any excluded phases, and also enables the component in the topology object.


```cpp
static void
hwloc_backend_disable(struct hwloc_backend *backend)
{
  if (backend->disable)
    backend->disable(backend);
  free(backend);
}

int
hwloc_backend_enable(struct hwloc_backend *backend)
{
  struct hwloc_topology *topology = backend->topology;
  struct hwloc_backend **pprev;

  /* check backend flags */
  if (backend->flags) {
    if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Cannot enable discovery component `%s' phases 0x%x with unknown flags %lx\n",
              backend->component->name, backend->component->phases, backend->flags);
    return -1;
  }

  /* make sure we didn't already enable this backend, we don't want duplicates */
  pprev = &topology->backends;
  while (NULL != *pprev) {
    if ((*pprev)->component == backend->component) {
      if (hwloc_components_verbose)
	fprintf(stderr, "hwloc: Cannot enable  discovery component `%s' phases 0x%x twice\n",
		backend->component->name, backend->component->phases);
      hwloc_backend_disable(backend);
      errno = EBUSY;
      return -1;
    }
    pprev = &((*pprev)->next);
  }

  if (hwloc_components_verbose)
    fprintf(stderr, "hwloc: Enabling discovery component `%s' with phases 0x%x (among 0x%x)\n",
	    backend->component->name, backend->phases, backend->component->phases);

  /* enqueue at the end */
  pprev = &topology->backends;
  while (NULL != *pprev)
    pprev = &((*pprev)->next);
  backend->next = *pprev;
  *pprev = backend;

  topology->backend_phases |= backend->component->phases;
  topology->backend_excluded_phases |= backend->component->excluded_phases;
  return 0;
}

```



这段代码是一个函数 `hwloc_backends_is_th磺系统` 和其他函数的一个成员。它的作用是判断当前硬件平台是否为所指定的系统，如果应用程序设置了 `set_foo()`，则可能会调用 `set_flags()` 来更新 `is_th磺系统` 标志。如果应用程序更改了后端，则可能需要使用 `HWLOC_TH磺系统` 环境变量。

具体来说，代码首先检查 `is_th磺系统` 是否已经设置，如果是，则将其设置为 0，否则将其设置为 1。接下来，代码遍历所有的后端，并检查其是否使用了 `set_foo()` 或 `set_flags()`。如果是，则代码将 `is_th磺系统` 更改为 0，否则继续遍历。接下来，代码使用 `topology->flags & HWLOC_TOPOLOGY_FLAG_IS_TH磺系统` 检查是否有 `HWLOC_TOPOLOGY_FLAG_IS_TH磺系统` 环境变量。如果有，则将其设置为 1。最后，代码再次遍历所有的后端，并检查其是否使用了 `set_foo()` 或 `set_flags()`。如果是，则代码将 `is_th磺系统` 更改为 0，否则继续遍历。


```cpp
void
hwloc_backends_is_thissystem(struct hwloc_topology *topology)
{
  struct hwloc_backend *backend;
  const char *local_env;

  /*
   * If the application changed the backend with set_foo(),
   * it may use set_flags() update the is_thissystem flag here.
   * If it changes the backend with environment variables below,
   * it may use HWLOC_THISSYSTEM envvar below as well.
   */

  topology->is_thissystem = 1;

  /* apply thissystem from normally-given backends (envvar_forced=0, either set_foo() or defaults) */
  backend = topology->backends;
  while (backend != NULL) {
    if (backend->envvar_forced == 0 && backend->is_thissystem != -1) {
      assert(backend->is_thissystem == 0);
      topology->is_thissystem = 0;
    }
    backend = backend->next;
  }

  /* override set_foo() with flags */
  if (topology->flags & HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM)
    topology->is_thissystem = 1;

  /* now apply envvar-forced backend (envvar_forced=1) */
  backend = topology->backends;
  while (backend != NULL) {
    if (backend->envvar_forced == 1 && backend->is_thissystem != -1) {
      assert(backend->is_thissystem == 0);
      topology->is_thissystem = 0;
    }
    backend = backend->next;
  }

  /* override with envvar-given flag */
  local_env = getenv("HWLOC_THISSYSTEM");
  if (local_env)
    topology->is_thissystem = atoi(local_env);
}

```

这段代码定义了一个名为 `hwloc_backends_find_callbacks` 的函数，属于 `hwloc_topology` 结构的成员函数。

该函数的主要作用是确定硬件布局中的后端(backend)并将其保存到 `topology` 结构中。通过调用 `backend` 对象的 `get_pci_busid_cpuset` 函数，该函数可以获取到后端的 ID。在循环中，函数先检查 `backend` 是否为 `NULL`，如果是，则设置 `topology` 结构中 `get_pci_busid_cpuset_backend` 函数的值为 `NULL`。否则，函数逐步执行下列操作：

1. 如果 `backend` 有 `get_pci_busid_cpuset` 函数，函数将 `topology` 结构中 `get_pci_busid_cpuset_backend` 函数的值设置为 `backend`。
2. 如果 `backend` 的 `next` 属性为 `NULL`，函数将 `backend` 指向的后续的后端(即 `backend` 的下一个节点)作为参数传入。
3. 如果 `backend` 有 `get_pci_busid_cpuset` 函数，但 `topology` 结构中 `get_pci_busid_cpuset_backend` 函数的值为 `NULL`，函数将 `backend` 指向并调用 `get_pci_busid_cpuset` 函数。

最终，函数返回。


```cpp
void
hwloc_backends_find_callbacks(struct hwloc_topology *topology)
{
  struct hwloc_backend *backend = topology->backends;
  /* use the first backend's get_pci_busid_cpuset callback */
  topology->get_pci_busid_cpuset_backend = NULL;
  while (backend != NULL) {
    if (backend->get_pci_busid_cpuset) {
      topology->get_pci_busid_cpuset_backend = backend;
      return;
    }
    backend = backend->next;
  }
  return;
}

```

这段代码定义了一个名为 `hwloc_backends_disable_all` 的函数，它的作用是遍历给定的 `hwloc_topology` 结构体中的所有后端，并将其父节点（即下一层）作为结束条件。在遍历过程中，函数会根据传入的 `hwloc_components_verbose` 参数打印出当前后端的名称。

函数的具体实现可以分为两个步骤：

1. 遍历给定的 topology 结构体，从后往前遍历所有后端，使用变量 `backend` 记录当前后端。
2. 对于每个后端，使用变量 `next` 记录后端的下一个后端，然后调用 `hwloc_backend_disable` 函数，将当前后端从 topology 中移除，并将 `topology->backends` 指向下一个后端。

最后，函数会将 topology 结构体中的所有后端指针设置为 NULL，以及将 topology 结构体中的后端包含的阶段数设置为 0。


```cpp
void
hwloc_backends_disable_all(struct hwloc_topology *topology)
{
  struct hwloc_backend *backend;

  while (NULL != (backend = topology->backends)) {
    struct hwloc_backend *next = backend->next;
    if (hwloc_components_verbose)
      fprintf(stderr, "hwloc: Disabling discovery component `%s'\n",
	      backend->component->name);
    hwloc_backend_disable(backend);
    topology->backends = next;
  }
  topology->backends = NULL;
  topology->backend_excluded_phases = 0;
}

```



这段代码是 hwloc_topology_components_fini 函数，它是 hwloc_topology 结构体的一部分，用于在遍历顶级头时释放被禁止的硬件组件。

在函数体中，首先使用 assert 语句来确保 topology 结构体中已经没有后端，因为如果是的话，后面所有针对后端的操作都将无法执行。然后，释放 topology 结构体中的 blacklisted_components 成员，这个成员数组可能是 topology 结构体通过其他函数进行初始化时被禁止的硬件组件的数组。最后，没有其他语句，函数结束。


```cpp
void
hwloc_topology_components_fini(struct hwloc_topology *topology)
{
  /* hwloc_backends_disable_all() must have been called earlier */
  assert(!topology->backends);

  free(topology->blacklisted_components);
}

```