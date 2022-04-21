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
- Node *lastfree: 
- GCObject *gclist: 跟GC有关
- sizearray: 数组部分的大小

### Node
~~~cpp
typedef struct Node {
  TValue i_val;
  TKey i_key;
} Node;
~~~

### Tvalue
~~~cpp
typedef TValue *StkId;  /* index to stack elements */

~~~


-----------------