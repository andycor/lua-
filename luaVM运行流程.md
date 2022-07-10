# lua虚拟机的主体运行流程
一个简单lua解释器的伪代码如下
~~~cpp
lua_State *L = luaL_newstate(); // 打开lua
luaL_dofile(L, filename) //加载文件并执行
lua_close(L); // 关闭lua
~~~
下面依次分析每一条语句。

# luaL_newState()
先来看lua_State和global_State这个两个数据结构
~~~cpp
/*
** `global state', shared by all threads of this state
用于存放全局变量
*/
typedef struct global_State {
  stringtable strt;  /* hash table for strings */// 存放所有字符串的容器
  lua_Alloc frealloc;  /* function to reallocate memory *///
  void *ud;         /* auxiliary data to `frealloc' */
  lu_byte currentwhite;
  lu_byte gcstate;  /* state of garbage collector */ //GC的状态
  int sweepstrgc;  /* position of sweep in `strt' */
  GCObject *rootgc;  /* list of all collectable objects *///所有需要GC的object都放在该链表
  GCObject **sweepgc;  /* position of sweep in `rootgc' */
  GCObject *gray;  /* list of gray objects */
  GCObject *grayagain;  /* list of objects to be traversed atomically */
  GCObject *weak;  /* list of weak tables (to be cleared) */
  GCObject *tmudata;  /* last element of list of userdata to be GC */// 所有有GC方法的udata都放在tmudata链表中
  Mbuffer buff;  /* temporary buffer for string concatentation */
  lu_mem GCthreshold; // 一个阈值，当这个totalbytes大于这个阈值时进行自动GC
  lu_mem totalbytes;  /* number of bytes currently allocated */  // 保存当前分配的总内存数量
  lu_mem estimate;  /* an estimate of number of bytes actually in use */  // 一个估算值，根据这个计算GCthreshold
  lu_mem gcdept;  /* how much GC is `behind schedule' */  // 当前待GC的数据大小，其实就是累加totalbytes和GCthreshold的差值
  int gcpause;  /* size of pause between successive GCs */  // 可以配置的一个值，不是计算出来的，根据这个计算GCthreshold，以此来控制下一次GC触发的时间
  int gcstepmul;  /* GC `granularity' */  // 每次进行GC操作回收的数据比例，见lgc.c/luaC_step函数
  lua_CFunction panic;  /* to be called in unprotected errors */
  TValue l_registry;
  struct lua_State *mainthread;
  UpVal uvhead;  /* head of double-linked list of all open upvalues */
  struct Table *mt[NUM_TAGS];  /* metatables for basic types */
  TString *tmname[TM_N];  /* array with tag-method names */
} global_State;
~~~

~~~cpp
/*
** `per thread' state
存放每个线程运行的时候状态、数据
*/
struct lua_State {
  CommonHeader;
  lu_byte status;
  StkId top;  /* first free slot in the stack */ //栈顶位置，也是当前栈的下一个可用位置
  StkId base;  /* base of current function */ //当前函数栈的基地址。
  global_State *l_G;// 指向全局变量的指针
  CallInfo *ci;  /* call info for current function */ //记录当前函数的执行信息
  const Instruction *savedpc;  /* `savedpc' of current function */ //当前函数的程序计数器，也是字节码
  StkId stack_last;  /* last free slot in the stack */ //栈中最后一个空位
  StkId stack;  /* stack base *///栈数组的起始位置
  CallInfo *end_ci;  /* points after end of ci array*/ //最后一个CallInfo的指针
  CallInfo *base_ci;  /* array of CallInfo's */ //存放CallInfo的数组
  int stacksize; //栈大小
  int size_ci;  /* size of array `base_ci' */ //CI数组大小
  unsigned short nCcalls;  /* number of nested C calls */
  unsigned short baseCcalls;  /* nested C calls when resuming coroutine */
  lu_byte hookmask;
  lu_byte allowhook;
  int basehookcount;
  int hookcount;
  lua_Hook hook;
  TValue l_gt;  /* table of globals */
  TValue env;  /* temporary place for environments */
  GCObject *openupval;  /* list of open upvalues in this stack */
  GCObject *gclist;
  struct lua_longjmp *errorJmp;  /* current error recover point */
  ptrdiff_t errfunc;  /* current error handling function (stack index) */
};
~~~

再来看luaL_newstate具体做了什么
~~~cpp
LUALIB_API lua_State *luaL_newstate (void) {
  lua_State *L = lua_newstate(l_alloc, NULL);
  if (L) lua_atpanic(L, &panic);
  return L;
}

LUA_API lua_State *lua_newstate (lua_Alloc f, void *ud) {
  int i;
  lua_State *L; 
  global_State *g;
  void *l = (*f)(ud, NULL, 0, state_size(LG)); //分配内存，LG=L+G
  if (l == NULL) return NULL;
  L = tostate(l); //转化类型为lua_State
  g = &((LG *)L)->g;
  //下面是初始化lua_State和global_State的内容，故省略
}

其实就是L（lua_State ）和G（global_State）分配内存以及初始化。

~~~
-------------------------------
# luaL_dofile 

~~~cpp
define luaL_dofile(L, fn) \
	(luaL_loadfile(L, fn) || lua_pcall(L, 0, LUA_MULTRET, 0))
~~~

由以上宏定义可知，luaL_dofile由两部分组成：
1.  luaL_loadfile() 加载.lua文件并做词法分析、语法分析。将代码转换成字节码。
2.  lua_pcall() 将字节码放到虚拟机中执行。

## luaL_loadfile()
luaL_loadfile的大致流程是 ：
1. luaL_loadfile()使用fopen读取代码文件并做预处理
2. lua_load() 创建并初始化ZIO ->   
3. luaD_protectedparser() 使用luaD_pcall执行代码解析函数f_parser来解析lua代码

f_parser函数如下：
~~~cpp
static void f_parser (lua_State *L, void *ud) { //代码分析器
  int i;
  Proto *tf; //函数
  Closure *cl; //闭包
  struct SParser *p = cast(struct SParser *, ud);
  int c = luaZ_lookahead(p->z);
  luaC_checkGC(L);
  tf = ((c == LUA_SIGNATURE[0]) ? luaU_undump : luaY_parser)(L, p->z,
                                                             &p->buff, p->name); // 通过luaY_parser分析代码得到tf(Proto)
  //以上没太搞清楚，等看完完整流程再来解答
  cl = luaF_newLclosure(L, tf->nups, hvalue(gt(L))); //创建closeure并分配内存
  cl->l.p = tf;
  for (i = 0; i < tf->nups; i++)  /* initialize eventual upvalues */
    cl->l.upvals[i] = luaF_newupval(L); ////创建upvalues并分配内存
  setclvalue(L, L->top, cl);//压入栈中
  incr_top(L);
}
~~~

Proto是分析阶段的产物，它包含了执行阶段所需要的各种数据，它是连接分析阶段和执行阶段的纽带，其具体数据结构如下：

~~~cpp
/*
** Function Prototypes 函数原型！
*/
typedef struct Proto {
  CommonHeader;
  TValue *k;  /* constants used by the function */ //常量表
  Instruction *code; //包含该函数实际调用的指令序列，类型是int32，一个int32整数就代表一条指令。
  struct Proto **p;  /* functions defined inside the function */ // 在这个函数中定义的函数，嵌套函数的原型数组
  int *lineinfo;  /* map from opcodes to source lines */
  struct LocVar *locvars;  /* information about local variables */ 
  TString **upvalues;  /* upvalue names */ //这里不是函数的上值，函数的上值存放在LClosure闭包中
  TString  *source; // 用于debug
  int sizeupvalues;
  int sizek;  /* size of `k' */
  int sizecode;
  int sizelineinfo;
  int sizep;  /* size of `p' */
  int sizelocvars;
  int linedefined;
  int lastlinedefined;
  GCObject *gclist;
  lu_byte nups;  /* number of upvalues */
  lu_byte numparams;
  lu_byte is_vararg;
  lu_byte maxstacksize;
} Proto;
~~~
这里不详细分析词法语法分析的过程。

--------------------------------------

## lua_pcall
lua_pcall的大致流程是
1. lua_pcall()
2. luaD_pcall()
3. luaD_call()
4. luaD_precall() & luaV_execute()  

重点是luaD_precall() 和 luaV_execute()，后者执行指令，前者完成执行指令前的准备工作。

### luaD_precall() 执行前准备
~~~cpp
int luaD_precall (lua_State *L, StkId func, int nresults) {
  LClosure *cl;
  ptrdiff_t funcr;
  if (!ttisfunction(func)) /* `func' is not a function? */
    func = tryfuncTM(L, func);  /* check the `function' tag method */
  funcr = savestack(L, func);
  cl = &clvalue(func)->l; // 拿到函数的闭包
  L->ci->savedpc = L->savedpc;
  if (!cl->isC) {  /* Lua function? prepare its call */ //如果是lua函数
    CallInfo *ci; //创建CallInfo ci，记录执行时的信息，有func,base,top三个栈指针
    StkId st, base;
    Proto *p = cl->p;
    luaD_checkstack(L, p->maxstacksize);
    func = restorestack(L, funcr);
    if (!p->is_vararg) {  /* no varargs? */ //varargs是什么？
      base = func + 1;
      if (L->top > base + p->numparams)
        L->top = base + p->numparams;
    }
    else {  /* vararg function */
      int nargs = cast_int(L->top - func) - 1;
      base = adjust_varargs(L, p, nargs);
      func = restorestack(L, funcr);  /* previous call may change the stack */
    }
    ci = inc_ci(L);  /* now `enter' new function */
    ci->func = func;
    L->base = ci->base = base; //记录当前函数的栈底
    ci->top = L->base + p->maxstacksize;//记录当前函数的栈顶，maxstacksize是在词法分析中算出
    lua_assert(ci->top <= L->stack_last);
    L->savedpc = p->code;  /* starting point *///将字节码赋值给savedpc
    ci->tailcalls = 0;
    ci->nresults = nresults;
    for (st = L->top; st < ci->top; st++) //将多余参数设为nil
      setnilvalue(st);
    L->top = ci->top;
    if (L->hookmask & LUA_MASKCALL) { 
      L->savedpc++;  /* hooks assume 'pc' is already incremented */
      luaD_callhook(L, LUA_HOOKCALL, -1);
      L->savedpc--;  /* correct 'pc' */
    }
    return PCRLUA;
  }
  else {  /* if is a C function, call it */ // 如果是C函数
    CallInfo *ci;
    int n;
    luaD_checkstack(L, LUA_MINSTACK);  /* ensure minimum stack size */
    ci = inc_ci(L);  /* now `enter' new function */
    ci->func = restorestack(L, funcr);
    L->base = ci->base = ci->func + 1;
    ci->top = L->top + LUA_MINSTACK;
    lua_assert(ci->top <= L->stack_last);
    ci->nresults = nresults;
    if (L->hookmask & LUA_MASKCALL)
      luaD_callhook(L, LUA_HOOKCALL, -1);
    lua_unlock(L);
    n = (*curr_func(L)->c.f)(L);  /* do the actual call */ //执行函数
    lua_lock(L);
    if (n < 0)  /* yielding? */
      return PCRYIELD;
    else {
      luaD_poscall(L, L->top - n);
      return PCRC;
    }
  }
}
~~~
从代码中可以看出precall主要做的是执行指令前的预处理，它对函数类型进行区分。   
如果是lua函数，做以下工作：
1. 从lua_State的CallInfo数组里取得一个全新的CallInfo结构体，设置它的func、base、top指针（因为马上要执行一个新的函数了）。
2. 从词法语法分析的结果closure中取出Proto结构体，这个Proto结构体中存放了字节码（p->code）。
3. 给新创建的CallInfo赋值，尤其是top指针和base指针，他们分别代表了该函数在栈中的上下区间。
4. 将字节码赋值给lua_State的savedpc字段。
5. 将多余的函数参数赋值为Nil。

如果是C函数，做以下工作：
1. 从lua_State的CallInfo数组里取得一个全新的CallInfo结构体，设置它的func、base、top指针。
2. 执行函数n = (*curr_func(L)->c.f)(L);
3. 如果执行完毕，则调用luaD_poscall恢复函数执行前的状态。


----------------------------------------------

### luaV_execute() 执行字节码
~~~cpp
void luaV_execute (lua_State *L, int nexeccalls) {
  LClosure *cl;
  StkId base;
  TValue *k;
  const Instruction *pc;
 reentry:  /* entry point */
  lua_assert(isLua(L->ci));
  pc = L->savedpc; //拿出字节码，pc也是程序计数器
  cl = &clvalue(L->ci->func)->l; // 拿到该函数的闭包
  base = L->base;
  k = cl->p->k; // 常量
  /* main loop of interpreter *///循环执行字节码
  for (;;) {
    const Instruction i = *pc++; //程序计数器++，同时取出对应地址的字节码，用32位int来表示
    StkId ra;  //表示操作数a在栈上的位置
    if ((L->hookmask & (LUA_MASKLINE | LUA_MASKCOUNT)) &&
        (--L->hookcount == 0 || L->hookmask & LUA_MASKLINE)) {
      traceexec(L, pc);
      if (L->status == LUA_YIELD) {  /* did hook yield? */
        L->savedpc = pc - 1;
        return;
      }
      base = L->base;
    }
    /* warning!! several calls may realloc the stack and invalidate `ra' */
    ra = RA(i); //解析字节码，取出操作数A
    lua_assert(base == L->base && L->base == L->ci->base);
    lua_assert(base <= L->top && L->top <= L->stack + L->stacksize);
    lua_assert(L->top == L->ci->top || luaG_checkopenop(i));
    switch (GET_OPCODE(i)) { //对不同的指令，执行不同的操作。
      case OP_MOVE: {
        setobjs2s(L, ra, RB(i));
        continue;
      }
      case OP_LOADK:
      ......以下省略
    }
  }
}
~~~
从代码可以看出execute就是一个大循环，循环执行各种不同的指令。   
注意在执行完n-1条指令后，最后一条指令会调用luaD_poscall来恢复函数执行前的环境。
~~~cpp
~~~

### 代码运行时的栈状态和CallInfo
![avatar](/images/执行函数时的栈状态.png)
其中CallInfo是存放函数运行时数据的辅助数据结构，定义如下：
~~~cpp
typedef struct CallInfo {
  StkId base;  /* base for this function */
  StkId func;  /* function index in the stack */
  StkId	top;  /* top for this function */
  const Instruction *savedpc;
  int nresults;  /* expected number of results from this function */
  int tailcalls;  /* number of tail calls lost under this entry */
} CallInfo;
~~~

lua采用基于寄存器的虚拟机实现，指令格式如下。
![avatar](/images/指令格式.jpg)