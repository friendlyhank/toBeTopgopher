在redis中列表键，发布和订阅、慢查询、监视器等都用到了链表，Redis服务器本身还是用链表来保存多个客户端的状态信息，以及使用链表来构建客户端输出缓冲区。

链表节点adlist.h/listNode
```c
typedef struct listNode {
    struct listNode *prev;//前置节点
    struct listNode *next;//后置节点
    void *value;//节点值
} listNode;
```
```c
typedef struct list {
    listNode *head;//表头节点
    listNode *tail;//表尾节点
    //节点值复制函数
    void *(*dup)(void *ptr);
    //节点值释放函数
    void (*free)(void *ptr);
    //节点值对比函数
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

## 链表操作
### 创建新链表
创建一个不包含任何节点的新链表
```c
list *listCreate(void)
{
    struct list *list;
	//分配内存
    if ((list = zmalloc(sizeof(*list))) == NULL)
        return NULL;
    //初始化属性
    list->head = list->tail = NULL;
    list->len = 0;
    list->dup = NULL;
    list->free = NULL;
    list->match = NULL;
    return list;
}
```

### 链表头部插入新节点
```c
list *listAddNodeHead(list *list, void *value)
{
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else {
        node->prev = NULL;
        node->next = list->head;
        list->head->prev = node;
        list->head = node;
    }
    list->len++;
    return list;
}
```

### 链表尾部插入新节点
```c
list *listAddNodeTail(list *list, void *value)
{
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else {
        node->prev = list->tail;
        node->next = NULL;
        list->tail->next = node;
        list->tail = node;
    }
    list->len++;
    return list;
}
```

### list结构宏实现的函数
```bash
/* Functions implemented as macros */
#define listLength(l) ((l)->len)  //返回链表长度
#define listFirst(l) ((l)->head) //返回链表的表头节点
#define listLast(l) ((l)->tail)  //返回链表的表尾节点
#define listPrevNode(n) ((n)->prev) //返回给定节点的前置节点
#define listNextNode(n) ((n)->next) //返回给定节点的后继节点
#define listNodeValue(n) ((n)->value)//返回给定节点的值

#define listSetDupMethod(l,m) ((l)->dup = (m))
#define listSetFreeMethod(l,m) ((l)->free = (m))
#define listSetMatchMethod(l,m) ((l)->match = (m))

#define listGetDupMethod(l) ((l)->dup)
#define listGetFreeMethod(l) ((l)->free)
#define listGetMatchMethod(l) ((l)->match)
```

## 迭代器
```c
typedef struct listIter {
    listNode *next;
    int direction;
} listIter;
```
 - listNode为具体某个节点信息
 - direction遍历方向  AL_START_HEAD为由头部向尾部遍历   AL_START_TAIL为由尾部向头部遍历

### 根据链表得到迭代器
```c
listIter *listGetIterator(list *list, int direction)
{
    listIter *iter;

    if ((iter = zmalloc(sizeof(*iter))) == NULL) return NULL;
    if (direction == AL_START_HEAD)
        iter->next = list->head;
    else
        iter->next = list->tail;
    iter->direction = direction;
    return iter;
}
```

### 释放迭代器
```c
void listReleaseIterator(listIter *iter) {
    zfree(iter);
}
```

```c
listNode *listNext(listIter *iter)
{
    listNode *current = iter->next;

    if (current != NULL) {
        if (iter->direction == AL_START_HEAD)
            iter->next = current->next;
        else
            iter->next = current->prev;
    }
    return current;
}
```

## 查询链表中和key匹配的节点
```c
listNode *listSearchKey(list *list, void *key)
{
    listIter iter;
    listNode *node;
	//生成由头部->尾部的迭代器
    listRewind(list, &iter);
    //迭代器开始遍历每个节点
    while((node = listNext(&iter)) != NULL) {
        if (list->match) {
            if (list->match(node->value, key)) {
                return node;
            }
        } else {
            if (key == node->value) {
                return node;
            }
        }
    }
    return NULL;
}
```

## list API
| 函数|作用|
|--|--|
| listCreate | 创建新链表 |
| listRelease | 释放给定链表，以及链表中的所有节点 |




