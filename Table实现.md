# Table

# 定义
~~~cpp
typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */ 
  lu_byte lsizenode;  /* log2 of size of `node' array */
  struct Table *metatable;
  TValue *array;  /* array part */
  Node *node;
  Node *lastfree;  /* any free position is before this position */
  GCObject *gclist;
  int sizearray;  /* size of `array' array */
} Table;
~~~
- flags: 
- lsizenode: 表中散列表部分的log2大小，用于数组和散列表的重新分配大小
- struct Table *metatable: 元表，直接复用了Table
- TValue array: 表中的数组部分
- Node *node: 应该是Hash table的头指针；Node结构见下文。
- Node *lastfree: 记录一个空闲位置的下一个位置。
- GCObject *gclist: 跟GC有关
- sizearray: 数组部分的大小

## Node
~~~cpp
typedef struct Node {
  TValue i_val;
  TKey i_key;
} Node;
~~~

## Tvalue
~~~cpp
TValue定义
typedef struct lua_TValue {
  TValuefields;
} TValue;

TValuefields 定义
#define TValuefields	Value value; int tt

Value定义
typedef union { //Union of all Lua values
  GCObject *gc;
  void *p;
  lua_Number n;
  int b;
} Value;
~~~
TValue是多种类型(union)中的一种，可能是GC、int、lua_Number或者任意类型。
所以lua table的value可以是任意值。

## TKey
~~~cpp
typedef union TKey {
  struct {
    TValuefields;
    struct Node *next;  /* for chaining */
  } nk;
  TValue tvk;
} TKey;
~~~
TKey也可以是多种类型，且多了一个next指针来实现链表。

-----------------

# 方法
介绍一些常用table方法的实现

## hashnum 
~~~cpp
/*
** hash for lua_Numbers
*/
static Node *hashnum (const Table *t, lua_Number n) {
  unsigned int a[numints];
  int i;
  if (luai_numeq(n, 0))  /* avoid problems with -0 */
    return gnode(t, 0);
  memcpy(a, &n, sizeof(a));
  for (i = 1; i < numints; i++) a[0] += a[i];
  return hashmod(t, a[0]);
}

tips:
#define numints		cast_int(sizeof(lua_Number)/sizeof(int))  //lua_Numer是double类型，所以numints=2
#define luai_numeq(a,b)		((a)==(b))
#define gnode(t,i)	(&(t)->node[i]) //取得hash值后，返回对应的node节点
#define hashmod(t,n)	(gnode(t, ((n) % ((sizenode(t)-1)|1)))) //hash方法，n是通过累计算出来的值，然后将其对散列域的大小取模，得到对应的下标？但是这样不需要处理冲突吗？
#define sizenode(t)	(twoto((t)->lsizenode)) //取表t的lsizenode值后，得到右移其值之后的二进制数。其值等于t的散列域的大小。
#define twoto(x)	(1<<(x))
~~~
代码意思是根据数字key返回相应的value
但是为什么要memcpy呢？
答：为了求hash值，所以做了累加的操作。（这样做效率真的高吗）

## mainposition
~~~cpp
/*
** returns the `main' position of an element in a table (that is, the index
** of its hash value) 
*/
static Node *mainposition (const Table *t, const TValue *key) {
  switch (ttype(key)) {
    case LUA_TNUMBER:
      return hashnum(t, nvalue(key));
    case LUA_TSTRING:
      return hashstr(t, rawtsvalue(key));
    case LUA_TBOOLEAN:
      return hashboolean(t, bvalue(key));
    case LUA_TLIGHTUSERDATA:
      return hashpointer(t, pvalue(key));
    default:
      return hashpointer(t, gcvalue(key));
  }
}
tips
其他hash方法就不详细看了
#define hashpow2(t,n)      (gnode(t, lmod((n), sizenode(t))))
#define hashstr(t,str)  hashpow2(t, (str)->tsv.hash)
#define hashboolean(t,p)        hashpow2(t, p)
#define hashpointer(t,p)	hashmod(t, IntPoint(p))
~~~
代码意思是根据key的种类来分类查询Table并返回。
仅key在散列域的时候使用，毕竟数组域不需要解析key。

## findindex
~~~cpp
/*
** returns the index of a `key' for table traversals. First goes all
** elements in the array part, then elements in the hash part. The
** beginning of a traversal is signalled by -1.

*/
static int findindex (lua_State *L, Table *t, StkId key) {
  int i;
  if (ttisnil(key)) return -1;  /* first iteration */ 判断key是否是nil
  i = arrayindex(key);
  if (0 < i && i <= t->sizearray)  /* is `key' inside array part? */判断是否在数组域
    return i-1;  /* yes; that's the index (corrected to C) */ i-=1为了适配C数组
  else {
    Node *n = mainposition(t, key);
    do {  /* check whether `key' is somewhere in the chain */ 遍历HASH链表。n是根据hash值找到的node,如果冲突了，即不是想要的node节点，则根据next找下去。
      /* key may be dead already, but it is ok to use it in `next' */
      if (luaO_rawequalObj(key2tval(n), key) ||
            (ttype(gkey(n)) == LUA_TDEADKEY && iscollectable(key) &&
             gcvalue(gkey(n)) == gcvalue(key))) {
        i = cast_int(n - gnode(t, 0));  /* key index in hash table */
        /* hash elements are numbered after array ones */
        return i + t->sizearray;这里要加上sizearray的原因是散列域的index是接着数组域增长的
      }
      else n = gnext(n);
    } while (n);
    luaG_runerror(L, "invalid key to " LUA_QL("next"));  /* key not found */没找到就报错。
    return 0;  /* to avoid warnings */
  }
}
~~~
用于表遍历，返回key值的index。

## luaH_new
~~~cpp
Table *luaH_new (lua_State *L, int narray, int nhash) {  创建一个新的表，narray：数组域大小，nhash：散列域大小
  Table *t = luaM_new(L, Table);  分配一个Table大小的空间
  luaC_link(L, obj2gco(t), LUA_TTABLE); 链接到GC的链表上
  t->metatable = NULL;
  t->flags = cast_byte(~0);
  /* temporary values (kept only if some malloc fails) */
  t->array = NULL;
  t->sizearray = 0;
  t->lsizenode = 0;
  t->node = cast(Node *, dummynode); 把dummynode转化为Node *类型
  setarrayvector(L, t, narray); 初始化数组域大小
  setnodevector(L, t, nhash); 初始化散列域大小
  return t;
}
tips
#define luaM_malloc(L,t)	luaM_realloc_(L, NULL, 0, (t))
#define luaM_new(L,t)		cast(t *, luaM_malloc(L, sizeof(t)))

#define dummynode		(&dummynode_)
static const Node dummynode_ = {
  {{NULL}, LUA_TNIL},  /* value */
  {{{NULL}, LUA_TNIL, NULL}}  /* key */
};
~~~

## newkey
~~~cpp
/*
** inserts a new key into a hash table; first, check whether key's main 
** position is free. If not, check whether colliding node is in its main 
** position or not: if it is not, move colliding node to an empty place and 
** put new key in its main position; otherwise (colliding node is in its main 
** position), new key goes to an empty position. 
*/
static TValue *newkey (lua_State *L, Table *t, const TValue *key) {
  Node *mp = mainposition(t, key);
  if (!ttisnil(gval(mp)) || mp == dummynode) { 如果找到的node的value不为空，冲突了
    Node *othern;
    Node *n = getfreepos(t);  /* get a free place */ 找到一个空闲的位置，具体见tips
    if (n == NULL) {  /* cannot find a free place? */
      rehash(L, t, key);  /* grow table */ 如果没有空闲位置，则重新洗牌
      return luaH_set(L, t, key);  /* re-insert key into grown table */ 重新插入一次，详见下文
    }
    lua_assert(n != dummynode);
    othern = mainposition(t, key2tval(mp)); 处理冲突，再找一个位置
    if (othern != mp) {  /* is colliding node out of its main position? */
      /* yes; move colliding node into free position */
      while (gnext(othern) != mp) othern = gnext(othern);  /* find previous */
      gnext(othern) = n;  /* redo the chain with `n' in place of `mp' */
      *n = *mp;  /* copy colliding node into free pos. (mp->next also goes) */
      gnext(mp) = NULL;  /* now `mp' is free */
      setnilvalue(gval(mp));
    }
    else {  /* colliding node is in its own main position */
      /* new node will go into free position */
      gnext(n) = gnext(mp);  /* chain new position */
      gnext(mp) = n;
      mp = n;
    }
  }
  gkey(mp)->value = key->value; gkey(mp)->tt = key->tt;
  luaC_barriert(L, t, key);
  lua_assert(ttisnil(gval(mp)));
  return gval(mp);
}
tips 
static Node *getfreepos (Table *t) { 找到一个空闲位置。
  while (t->lastfree-- > t->node) {  t-lastfree-- 表示记录一个空闲位置。--表示可以向前遍历，直到头指针的位置。
    if (ttisnil(gkey(t->lastfree)))
      return t->lastfree;
  }
  return NULL;  /* could not find a free place */
}
~~~
插入一个新key


烂尾了，感觉不如直接把注释写在源码上






















