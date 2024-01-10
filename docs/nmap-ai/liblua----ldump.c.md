# `nmap\liblua\ldump.c`

```
# 定义了一个结构体 DumpState，用于保存状态信息
typedef struct {
  lua_State *L;  # Lua 状态机
  lua_Writer writer;  # 写入函数指针
  void *data;  # 数据指针
  int strip;  # 是否剥离调试信息
  int status;  # 状态信息
} DumpState;

# dumpBlock 函数，用于将数据块写入到状态中
static void dumpBlock (DumpState *D, const void *b, size_t size) {
  if (D->status == 0 && size > 0) {  # 如果状态为 0 并且数据大小大于 0
    lua_unlock(D->L);  # 解锁 Lua 状态机
    D->status = (*D->writer)(D->L, b, size, D->data);  # 调用写入函数指针，将数据写入状态中
    lua_lock(D->L);  # 锁定 Lua 状态机
  }
}

# dumpVar 函数，用于将变量写入到状态中
#define dumpVar(D,x)        dumpVector(D,&x,1)

# dumpByte 函数，将整数转换为字节并写入状态中
static void dumpByte (DumpState *D, int y) {
  lu_byte x = (lu_byte)y;  # 将整数转换为字节
  dumpVar(D, x);  # 将字节写入状态中
}

# dumpSize 函数，将大小写入状态中
static void dumpSize (DumpState *D, size_t x) {
  lu_byte buff[DIBS];  # 定义一个字节缓冲区
  int n = 0;  # 初始化 n 为 0
  do {
    buff[DIBS - (++n)] = x & 0x7f;  # 将 x 的低 7 位存入缓冲区，逆序填充缓冲区
    x >>= 7;  # x 右移 7 位
  } while (x != 0);  # 当 x 不为 0 时循环
  buff[DIBS - 1] |= 0x80;  # 标记最后一个字节
  dumpVector(D, buff + DIBS - n, n);  # 将缓冲区中的数据写入状态中
}

# dumpInt 函数，将整数写入状态中
static void dumpInt (DumpState *D, int x) {
  dumpSize(D, x);  # 调用 dumpSize 函数，将整数写入状态中
}

# dumpNumber 函数，将 Lua 数字类型写入状态中
static void dumpNumber (DumpState *D, lua_Number x) {
  dumpVar(D, x);  # 将 Lua 数字写入状态中
}

# dumpInteger 函数，将 Lua 整数类型写入状态中
static void dumpInteger (DumpState *D, lua_Integer x) {
  dumpVar(D, x);  # 将 Lua 整数写入状态中
}

# dumpString 函数，将字符串写入状态中
static void dumpString (DumpState *D, const TString *s) {
  if (s == NULL)  # 如果字符串为空
    dumpSize(D, 0);  # 将大小为 0 写入状态中
  else {
    size_t size = tsslen(s);  # 获取字符串长度
    const char *str = getstr(s);  # 获取字符串内容
    dumpSize(D, size + 1);  # 将字符串大小加 1 写入状态中
    dumpVector(D, str, size);  # 将字符串内容写入状态中
  }
}

# dumpCode 函数，将函数的指令集写入状态中
static void dumpCode (DumpState *D, const Proto *f) {
  dumpInt(D, f->sizecode);  # 将指令集大小写入状态中
  dumpVector(D, f->code, f->sizecode);  # 将指令集内容写入状态中
}

# dumpFunction 函数，将函数信息写入状态中
static void dumpFunction(DumpState *D, const Proto *f, TString *psource);
static void dumpConstants (DumpState *D, const Proto *f) {
  // 获取常量表的大小并进行转储
  int i;
  int n = f->sizek;
  dumpInt(D, n);
  // 遍历常量表
  for (i = 0; i < n; i++) {
    const TValue *o = &f->k[i];
    int tt = ttypetag(o);
    // 转储常量的类型标记
    dumpByte(D, tt);
    switch (tt) {
      // 如果是浮点数类型的常量，转储浮点数值
      case LUA_VNUMFLT:
        dumpNumber(D, fltvalue(o));
        break;
      // 如果是整数类型的常量，转储整数值
      case LUA_VNUMINT:
        dumpInteger(D, ivalue(o));
        break;
      // 如果是短字符串或长字符串类型的常量，转储字符串值
      case LUA_VSHRSTR:
      case LUA_VLNGSTR:
        dumpString(D, tsvalue(o));
        break;
      // 如果是空值、假值或真值类型的常量，进行断言确认
      default:
        lua_assert(tt == LUA_VNIL || tt == LUA_VFALSE || tt == LUA_VTRUE);
    }
  }
}

static void dumpProtos (DumpState *D, const Proto *f) {
  // 获取原型函数的数量并进行转储
  int i;
  int n = f->sizep;
  dumpInt(D, n);
  // 遍历原型函数
  for (i = 0; i < n; i++)
    // 转储原型函数
    dumpFunction(D, f->p[i], f->source);
}

static void dumpUpvalues (DumpState *D, const Proto *f) {
  // 获取Upvalue的数量并进行转储
  int i, n = f->sizeupvalues;
  dumpInt(D, n);
  // 遍历Upvalue
  for (i = 0; i < n; i++) {
    // 转储Upvalue的instack标记
    dumpByte(D, f->upvalues[i].instack);
    // 转储Upvalue的idx
    dumpByte(D, f->upvalues[i].idx);
    // 转储Upvalue的kind
    dumpByte(D, f->upvalues[i].kind);
  }
}

static void dumpDebug (DumpState *D, const Proto *f) {
  int i, n;
  // 如果strip标记为真，则将n设为0，否则获取行信息的大小并进行转储
  n = (D->strip) ? 0 : f->sizelineinfo;
  dumpInt(D, n);
  // 转储行信息
  dumpVector(D, f->lineinfo, n);
  // 如果strip标记为真，则将n设为0，否则获取绝对行信息的大小并进行转储
  n = (D->strip) ? 0 : f->sizeabslineinfo;
  dumpInt(D, n);
  // 遍历绝对行信息
  for (i = 0; i < n; i++) {
    // 转储绝对行信息的pc
    dumpInt(D, f->abslineinfo[i].pc);
    // 转储绝对行信息的line
    dumpInt(D, f->abslineinfo[i].line);
  }
  // 如果strip标记为真，则将n设为0，否则获取局部变量信息的大小并进行转储
  n = (D->strip) ? 0 : f->sizelocvars;
  dumpInt(D, n);
  // 遍历局部变量信息
  for (i = 0; i < n; i++) {
    // 转储局部变量的名称
    dumpString(D, f->locvars[i].varname);
    // 转储局部变量的起始pc
    dumpInt(D, f->locvars[i].startpc);
    // 转储局部变量的结束pc
    dumpInt(D, f->locvars[i].endpc);
  }
  // 如果strip标记为真，则将n设为0，否则获取Upvalue的数量并进行转储
  n = (D->strip) ? 0 : f->sizeupvalues;
  dumpInt(D, n);
  // 遍历Upvalue
  for (i = 0; i < n; i++)
    // 转储Upvalue的名称
    dumpString(D, f->upvalues[i].name);
}

static void dumpFunction (DumpState *D, const Proto *f, TString *psource) {
  // 如果strip标记为真或者函数的源与其父函数的源相同，则转储空字符串
  if (D->strip || f->source == psource)
    dumpString(D, NULL);  /* no debug info or same source as its parent */
  else
  # 将字符串数据写入到指定的输出流中
  dumpString(D, f->source);
  # 将整数数据写入到指定的输出流中，表示函数定义所在的起始行
  dumpInt(D, f->linedefined);
  # 将整数数据写入到指定的输出流中，表示函数定义所在的结束行
  dumpInt(D, f->lastlinedefined);
  # 将字节数据写入到指定的输出流中，表示函数的参数个数
  dumpByte(D, f->numparams);
  # 将字节数据写入到指定的输出流中，表示函数是否具有可变参数
  dumpByte(D, f->is_vararg);
  # 将字节数据写入到指定的输出流中，表示函数所需的最大寄存器数量
  dumpByte(D, f->maxstacksize);
  # 将函数的指令集写入到指定的输出流中
  dumpCode(D, f);
  # 将函数的常量数据写入到指定的输出流中
  dumpConstants(D, f);
  # 将函数的上值数据写入到指定的输出流中
  dumpUpvalues(D, f);
  # 将函数的子函数数据写入到指定的输出流中
  dumpProtos(D, f);
  # 将函数的调试信息写入到指定的输出流中
  dumpDebug(D, f);
# 以静态方式输出函数头部信息
static void dumpHeader (DumpState *D) {
  # 输出 Lua 签名
  dumpLiteral(D, LUA_SIGNATURE);
  # 输出 Lua 版本号
  dumpByte(D, LUAC_VERSION);
  # 输出 Lua 格式号
  dumpByte(D, LUAC_FORMAT);
  # 输出 Lua 数据
  dumpLiteral(D, LUAC_DATA);
  # 输出指令的大小
  dumpByte(D, sizeof(Instruction));
  # 输出整数的大小
  dumpByte(D, sizeof(lua_Integer));
  # 输出浮点数的大小
  dumpByte(D, sizeof(lua_Number));
  # 输出整数标志
  dumpInteger(D, LUAC_INT);
  # 输出浮点数标志
  dumpNumber(D, LUAC_NUM);
}

# 将 Lua 函数转换为预编译的代码块
int luaU_dump(lua_State *L, const Proto *f, lua_Writer w, void *data,
              int strip) {
  # 创建 DumpState 对象
  DumpState D;
  # 设置 Lua 状态
  D.L = L;
  # 设置写入函数
  D.writer = w;
  # 设置数据
  D.data = data;
  # 设置是否剥离
  D.strip = strip;
  # 设置状态为 0
  D.status = 0;
  # 输出函数头部信息
  dumpHeader(&D);
  # 输出函数的 upvalue 大小
  dumpByte(&D, f->sizeupvalues);
  # 输出函数信息
  dumpFunction(&D, f, NULL);
  # 返回状态
  return D.status;
}
```