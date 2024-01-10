# `nmap\nse_main.h`

```
#ifndef NMAP_LUA_H
#define NMAP_LUA_H

#include <vector>
#include <set>
#include <string>

#include "nse_lua.h"

#include "scan_lists.h"

class ScriptResult
{
  private:
    const char *id;  // 脚本结果的标识符
    /* 结构化输出表，存储在 L_NSE[LUA_REGISTRYINDEX] 中的整数引用 */
    int output_ref;
  public:
    ScriptResult() : id(NULL), output_ref(LUA_NOREF) {}  // 构造函数，初始化 id 和 output_ref
    ~ScriptResult() {
      // 确保释放 Lua 引用
      clear();
    }
    void clear (void);  // 清空脚本结果
    void set_output_tab (lua_State *, int);  // 设置输出表
    std::string get_output_str (void) const;  // 获取输出字符串
    const char *get_id (void) const { return id; }  // 获取标识符
    void write_xml() const;  // 写入 XML
    bool operator<(ScriptResult const &b) const {  // 重载小于运算符
      return strcmp(this->id, b.id) < 0;
    }
};

typedef std::multiset<ScriptResult *> ScriptResults;  // 定义 ScriptResults 类型为 ScriptResult 指针的多重集合

/* 调用此函数以获取一个 ScriptResults 对象，用于存储预扫描和后扫描脚本结果 */
ScriptResults *get_script_scan_results_obj (void);

class Target;


/* API */
int nse_yield (lua_State *, lua_KContext, lua_KFunction);  // NSE yield 函数
void nse_restore (lua_State *, int);  // NSE 恢复函数
void nse_destructor (lua_State *, char);  // NSE 析构函数
void nse_base (lua_State *);  // NSE 基础函数
void nse_selectedbyname (lua_State *);  // NSE 通过名称选择函数
void nse_gettarget (lua_State *, int);  // NSE 获取目标函数

void open_nse (void);  // 打开 NSE
void script_scan (std::vector<Target *> &targets, stype scantype);  // 执行脚本扫描
void close_nse (void);  // 关闭 NSE

#define SCRIPT_ENGINE "NSE"  // 定义脚本引擎为 "NSE"

#ifdef WIN32
#  define SCRIPT_ENGINE_LUA_DIR "scripts\\"  // Windows 下脚本目录
#  define SCRIPT_ENGINE_LIB_DIR "nselib\\"  // Windows 下库目录
#else
#  define SCRIPT_ENGINE_LUA_DIR "scripts/"  // 非 Windows 下脚本目录
#  define SCRIPT_ENGINE_LIB_DIR "nselib/"  // 非 Windows 下库目录
#endif

#define SCRIPT_ENGINE_DATABASE SCRIPT_ENGINE_LUA_DIR "script.db"  // 脚本引擎数据库
#define SCRIPT_ENGINE_EXTENSION ".nse"  // 脚本引擎扩展名

#endif
```