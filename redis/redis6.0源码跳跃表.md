> 跳跃表主要用于有序集合键，另一个是在集群节点中用作内部数据结构。
## 结构体定义
跳跃表的结构定义在server.h/zskiplist 
```c
typedef struct zskiplistNode {
    sds ele;//成员对象
    double score;//分数
    struct zskiplistNode *backward;
    //层
    struct zskiplistLevel {
    	//前进指针
        struct zskiplistNode *forward;
        //跨度
        unsigned long span;
    } level[];
} zskiplistNode;
```

```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```
 - header指向跳跃表的表头节点
 - tail指向跳跃表的表尾节点
 - level记录跳跃表内，层数最大的那个节点的层数
 - length跳跃表的长度

## 创建一个新的跳跃表
创建跳跃表函数在t_zset.c/zslCreate
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200514195015817.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
```c
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}
```

```c
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    //初始化跳跃表节点
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    //初始化最大层级为32
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

## 添加新节点到跳跃表
添加新节点到跳跃表是调用zslInsert函数实现的，我们将配合例子分段讲解。
假设我们要在已有跳跃表中插入新的节点29,跳跃表示例如图表示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200514222122852.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    .....
}
```
步骤一:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200514223014221.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 1. 由最高层开始遍历，i不断递减
 2. 如果前进指针不为空并且前进指针的分数小于插入节点的分数或者还有一种情况是两者分数相等，则需要条件是前进指针的字符串长度小于插入节点的字符串长度。
 3. 满足第二步条件会累加该层级的每个跳跃节点的span值，得到每个层级的rank[i]和新插入节点的前驱如图标红的update[i]。

步骤二：
```c
.....
	//计算出新节点的层高
	level = zslRandomLevel();
	//如果新节点的层高大于跳跃表当前高度
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;//为了方便计算新节点到尾部节点的距离
        }
        zsl->level = level;
    }
.....
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200515081838547.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 1. 得到第一步每层的update[i]和rank[i]之后，我们知道了插入的位置，如图虚线部分。
 2. 通过zslRandomLevel计算出新插入节点的层高,假设层高level为5
 3. 因为层级为5已经超出跳跃表的层级5，所以同样的我们要在L5计算出rank[i]和update[i],可以得到rank[4]=0,update[4]为header节点。
 4. 最后更新一下跳跃表最高层级为5。

步骤三：
```c
.....
	x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200518101312916.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 1. 新插入节点的前进指针等于新插入节点的前驱的前进指针
 2. 新插入节点的前驱的前进指针改为指向新插入节点
 3. 新节点的span=新插入节点前驱的span-(rank[0]-rank[i])
 4. 新插入节点前驱的span=(rank[0] - rank[i]) + 1
 5. 更新插入节点的后退指针
 
 每插入一个节点需要修改的部分为新插入节点的前驱节点的前进指针指向和span,还有新插入节点自身的前进指针指向和span,这里和链表类似。如果对3、4号不是特别理解，可以看如下图理解,比如在第四层，我们要计算出新插入节点的span，和新插入节点的前驱的span。
 
```c
x->level[3].span =update[3]->level[3].span - (rank[0] - rank[i])= 3-(4-2)=1
update[3]->level[3].span =(rank[0] - rank[3]) + 1= (4-2)+1=3
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200518081801925.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

## 删除节点
删除节点我们配合例子分段讲解。
假设我们要在已有跳跃表中删除分值为29的节点,跳跃表示例如图表示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200518101403507.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
```c
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    //找到每个层级要删除的前驱
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}
```
 1. 找到每个层级要删除节点的前驱,如图标红部分：
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200518101441899.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 2. 最底层L1要删除节点的前驱的前进指针就是可能要删除的节点,表示为x = x->level[0].forward，分值相等并且字符串长度相等则表示是要删除的节点数据。
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200518095337194.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 3. 更新每层要删除节点的前驱的span和forward,更新最底层L1的后退指针,具体的方法为zslDeleteNode。

```c
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    //更新每个层级的update[i]的span和forward
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }
    //更新最底层的后退指针
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    zsl->length--;
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200518101532934.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200518101844384.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
## 修改节点
修改节点的方法为zslUpdateScore，其实就是删除后重新插入的结果，所以这里不再重复讲解。

## 获得排位
给定分数和成员，计算出排位
```c
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) <= 0))) {
            rank += x->level[i].span;
            x = x->level[i].forward;
        }

        /* x might be equal to zsl->header, so test if obj is non-NULL */
        if (x->ele && sdscmp(x->ele,ele) == 0) {
            return rank;
        }
    }
    return 0;
}
```

更多讲解,欢迎关注我的github:
[go成神之路](https://github.com/friendlyhank/toBeTopgopher)

