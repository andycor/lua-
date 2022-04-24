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
  StkId top;  /* first free slot in the stack */
  StkId base;  /* base of current function */
  global_State *l_G; // 指向全局变量的指针
  CallInfo *ci;  /* call info for current function */
  const Instruction *savedpc;  /* `savedpc' of current function */
  StkId stack_last;  /* last free slot in the stack */
  StkId stack;  /* stack base */
  CallInfo *end_ci;  /* points after end of ci array*/
  CallInfo *base_ci;  /* array of CallInfo's */
  int stacksize;
  int size_ci;  /* size of array `base_ci' */
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

~~~
-------------------------------
# luaL_dofile 
~~~cpp
#define luaL_dofile(L, fn) \
	(luaL_loadfile(L, fn) || lua_pcall(L, 0, LUA_MULTRET, 0))
~~~
由以上宏定义可知，luaL_dofile由两部分组成：
1.  luaL_loadfile() 加载.lua文件并做词法分析、语法分析。将代码转换成字节码。
2.  lua_pcall() 将字节码放到虚拟机中执行。

## luaL_loadfile()
大致流程是  
luaL_loadfile()使用fopen读取代码文件并做预处理 ->   
lua_load() 创建并初始化ZIO ->   
luaD_protectedparser() 使用luaD_pcall执行代码解析函数f_parser   
~~~cpp
static void f_parser (lua_State *L, void *ud) { //代码分析器
  int i;
  Proto *tf; //函数
  Closure *cl; //闭包
  struct SParser *p = cast(struct SParser *, ud);
  int c = luaZ_lookahead(p->z);
  luaC_checkGC(L);
  tf = ((c == LUA_SIGNATURE[0]) ? luaU_undump : luaY_parser)(L, p->z,
                                                             &p->buff, p->name);
  //以上没太搞清楚，等看完完整流程再来解答
  cl = luaF_newLclosure(L, tf->nups, hvalue(gt(L))); //创建closeure并分配内存
  cl->l.p = tf;
  for (i = 0; i < tf->nups; i++)  /* initialize eventual upvalues */
    cl->l.upvals[i] = luaF_newupval(L); ////创建upvalues并分配内存
  setclvalue(L, L->top, cl);//压入栈中
  incr_top(L);
}
~~~
TODO 词法分析，语法分析的内容待填充

--------------------------------------

## lua_pcall
大致流程是
lua_pcall() ->  
luaD_pcall() ->  
luaD_call() ->  
luaD_precall() & luaV_execute()  

重点是luaD_precall() 和 luaV_execute()，后者执行指令，前者完成执行指令前的准备工作。
