---
title: lua数据结构table的键值存取过程源码分析
date: 2018-04-20
tags: [lua]
categories: 赖钧
description: table的键值存取源码分析
---

## 一、相关数据结构定义
### 1、Table结构定义
``` c
typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */ 
  lu_byte lsizenode;  /* log2 of size of `node' array */
  struct Table *metatable;
  TValue *array;  /* array part */
  Node *node;     /* hash part */
  Node *lastfree;  /* any free position is before this position */
  GCObject *gclist;
  int sizearray;  /* size of `array' array */
} Table;
```
可以看出array和node作为存储数据的两种结构，当table以整型作为key时,会用array来存储，其他数据类型key会用node来存储。sizearray作为数组的长度，2的lsizenode次方作为哈希表的长度。
### 2、TValue结构定义
``` c
/*
** Union of all Lua values
*/
typedef union {
  GCObject *gc;
  void *p;
  lua_Number n;
  int b;
} Value;

#define TValuefields	Value value; int tt

typedef struct lua_TValue {
  TValuefields;
} TValue;
```
TValue是一个结构体，里面是TValuefields，它由一个宏定义完成，包括Value和一个int值。 tt用于表示lua的基本数据类型，Value是一个联合体,可以是lua所支持的数据类型中的任意一种。

### 3、Node结构定义
``` c
typedef struct Node {
  TValue i_val;
  TKey i_key;
} Node;

typedef union TKey {
  struct {
    TValuefields;
    struct Node *next;  /* for chaining */
  } nk;
  TValue tvk;
} TKey;
```
Node是key-value对的结构，主要是TKey结构,是一个union，所以TKey的大小就是nk的大小，从TValue的定义可知TValue和TValuefields同一个结构，所以tvk和nk的TValuefields都表示键值。struct Node *next这个指针指向了下一个冲突(哈希碰撞)的node。

## 二、键值存取过程
### 1、键值的取值
``` c
const TValue *luaH_get (Table *t, const TValue *key) {
  switch (ttype(key)) {
    case LUA_TNIL: return luaO_nilobject;
    case LUA_TSTRING: return luaH_getstr(t, rawtsvalue(key));
    case LUA_TNUMBER: {
      int k;
      lua_Number n = nvalue(key);
      lua_number2int(k, n);
      if (luai_numeq(cast_num(k), nvalue(key))) /* index is int? */
        return luaH_getnum(t, k);  /* use specialized version */
      /* else go through */
    }
    default: {
      Node *n = mainposition(t, key);
      do {  /* check whether `key' is somewhere in the chain */
        if (luaO_rawequalObj(key2tval(n), key))
          return gval(n);  /* that's it */
        else n = gnext(n);
      } while (n);
      return luaO_nilobject;
    }
  }
}
```
首先对key值进行类型判断，如果是空那就返回空，是字符串就调用luaH_getstr（t，k），是数字则根据是否是整形调用luaH_getnum(t, k)，否则计算key的存储主位置，然后遍历Node节点寻找key对应的节点。其实luaH_getstr（t，k）和luaH_getnum(t, k)方法内部也是通过遍历寻找节点。
1.若是字符串LUA_TSTRING类型，则调用luaH_getstr（t，k)查找，方法如下

``` c
/*
** search function for strings
*/
const TValue *luaH_getstr (Table *t, TString *key) {
  Node *n = hashstr(t, key);
  do {  /* check whether `key' is somewhere in the chain */
    if (ttisstring(gkey(n)) && rawtsvalue(gkey(n)) == key)
      return gval(n);  /* that's it */
    else n = gnext(n);
  } while (n);
  return luaO_nilobject;
}
```
若key为字符串，则根据hashstr（t，key）方法找到该键对应的node位置，遍历链表，返回对应的值。
2.若是LUA_TNUMBER数字类型，则调用luaH_getnum(t, k)查找，方法如下:

``` c
/*
** search function for integers
*/
const TValue *luaH_getnum (Table *t, int key) {
  /* (1 <= key && key <= t->sizearray) */
  if (cast(unsigned int, key-1) < cast(unsigned int, t->sizearray))
    return &t->array[key-1];
  else {
    lua_Number nk = cast_num(key);
    Node *n = hashnum(t, nk);
    do {  /* check whether `key' is somewhere in the chain */
      if (ttisnumber(gkey(n)) && luai_numeq(nvalue(gkey(n)), nk))
        return gval(n);  /* that's it */
      else n = gnext(n);
    } while (n);
    return luaO_nilobject;
  }
}
```
若key大于等于1，且小于等于数组长度，则从数组中取出对应的键值。否则通过hashnum(t, nk)函数找到key对应的Node位置，然后遍历寻找对应的值。
3.若是其他类型，即不是空，数字，字符串类型，都是通过hash函数计算出节点存储主位置,然后遍历链表找查找（因为存在hash冲突的键值，冲突的键值在hash表中有Node *next指针链接起来）。
mainposition函数实现如下：

``` c
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
```
可以看出，此函数根据key值的类型选择不同的hash函数进行计算节点存储主位置
### 2、键值的存值
``` c
TValue *luaH_set (lua_State *L, Table *t, const TValue *key) {
  const TValue *p = luaH_get(t, key);
  t->flags = 0;
  if (p != luaO_nilobject)
    return cast(TValue *, p);
  else {
    if (ttisnil(key)) luaG_runerror(L, "table index is nil");
    else if (ttisnumber(key) && luai_numisnan(nvalue(key)))
      luaG_runerror(L, "table index is NaN");
    return newkey(L, t, key);
  }
}
```
首先判断key是否在table中，如果在就替换，否则调用newkey(L, t, key)插入新的key-value结构。
函数newkey代码如下:

``` c
/*
** inserts a new key into a hash table; first, check whether key's main 
** position is free. If not, check whether colliding node is in its main 
** position or not: if it is not, move colliding node to an empty place and 
** put new key in its main position; otherwise (colliding node is in its main 
** position), new key goes to an empty position. 
*/
static TValue *newkey (lua_State *L, Table *t, const TValue *key) {
  Node *mp = mainposition(t, key);
  if (!ttisnil(gval(mp)) || mp == dummynode) {
    Node *othern;
    Node *n = getfreepos(t);  /* get a free place */
    if (n == NULL) {  /* cannot find a free place? */
      rehash(L, t, key);  /* grow table */
      return luaH_set(L, t, key);  /* re-insert key into grown table */
    }
    lua_assert(n != dummynode);
    othern = mainposition(t, key2tval(mp));
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
```
从注释可以看出插入表的过程是，检查主位置（mainposition）是否为空，如果不为空则检查冲突Node是否在这个主位置。如果不在，移冲突Node到一个空位置，存放新Key在这个主位置，否则新Key存放到一个空位置。
在插入Key过程中，如果hash表空间已经满了，需要调用rehash函数增加表存储空间。
rehash函数实现如下：

``` c
static void rehash (lua_State *L, Table *t, const TValue *ek) {
  int nasize, na;
  int nums[MAXBITS+1];  /* nums[i] = number of keys between 2^(i-1) and 2^i */
  int i;
  int totaluse;
  for (i=0; i<=MAXBITS; i++) nums[i] = 0;  /* reset counts */
  nasize = numusearray(t, nums);  /* count keys in array part */
  totaluse = nasize;  /* all those keys are integer keys */
  totaluse += numusehash(t, nums, &nasize);  /* count keys in hash part */
  /* count extra key */
  nasize += countint(ek, nums);
  totaluse++;
  /* compute new size for array part */
  na = computesizes(nums, &nasize);
  /* resize the table to new computed sizes */
  resize(L, t, nasize, totaluse - na);
}
```
rehash首先统计当前table中到底有value值不是nil的键值对的个数，然后根据这个数值确定table中数组部分的大小（其大小保证数组部分的空间利用率必须50%），最后调用luaH_resize函数来重建table。 
具体过程是首先调用函数numusearray计算table中数组部分非nil的数值的个数，然后调用numusehash函数计算table中哈希部分的非nil的键值对的个数。调用countint函数来确定将要插入的（key,value）是否可以放在数组部分，接着调用computesizes来计算新的table数组部分的大小，最后调用luaH_resize函数根据原来table中数据构建新的table。

原文参考链接 https://blog.csdn.net/zr339361504/article/details/52432163

小知识：位运算的求余公式(该方法只对2的N次方数系有效) X & (2^N - 1)

