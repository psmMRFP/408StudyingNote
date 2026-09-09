# 链表：通过指针连接分散的节点

## 核心模型

> **链表不是物理上连续的一串数据，而是一组可以分散存放的节点；每个节点通过指针记录其他节点的位置。**

```text
顺序表：连续内存 + 下标计算地址
链表：  当前节点 + 从指针读取下一节点地址
```

## 1. 单链表节点在机器中是什么

```cpp
struct LNode {
    int data;
    LNode *next;
};
```

逻辑结构：

```text
一个节点
+--------------+------------------+
| data         | next             |
| 当前元素数据 | 下一节点的地址   |
+--------------+------------------+
```

假设某节点的内容是：

```text
节点自身地址：0x2000

data = 20
next = 0x7000
```

它表示：

```text
当前节点保存数据 20
下一节点位于地址 0x7000
```

不表示下一节点物理上紧挨当前节点。

### 地址模型的边界

程序中看到的指针值通常是**进程虚拟地址**。CPU和操作系统还要通过页表等机制将虚拟地址转换成物理地址。数据结构题把这部分抽象掉，只关心：

```text
next 是否为空
next 是否正确指向逻辑后继
```

节点也不一定只能来自堆；它可以来自静态区、对象池或其他合法存储区。常规链表实现使用动态分配，是因为节点需要按需创建和释放。

## 2. 节点可以分散在内存中

逻辑顺序：

```text
10 -> 20 -> 30
```

一种可能的内存状态：

```text
地址 0x9000
+----------+----------+
| data=10  | 0x2000   |
+----------+----------+

地址 0x2000
+----------+----------+
| data=20  | 0x7000   |
+----------+----------+

地址 0x7000
+----------+----------+
| data=30  | nullptr  |
+----------+----------+
```

CPU执行遍历时，程序的指针状态依次为：

```text
p = 0x9000
读取 p->data = 10
读取 p->next = 0x2000

p = 0x2000
读取 p->data = 20
读取 p->next = 0x7000

p = 0x7000
读取 p->data = 30
读取 p->next = nullptr

停止
```

## 3. 为什么不能 O(1) 访问第 i 个节点

顺序表可以根据首地址、下标和元素大小直接计算地址。链表通常只有第一个节点的地址，无法从位序直接算出后续节点地址。

要访问第4个数据节点，只能读取3次 `next`：

```text
第1个节点
    |
    | 读取 next
    v
第2个节点
    |
    | 读取 next
    v
第3个节点
    |
    | 读取 next
    v
第4个节点
```

访问第 `i` 个节点需要沿链移动 `i - 1` 次，最坏访问表尾为 `O(n)`。

> 顺序表靠“计算地址”；链表靠“读取当前节点中保存的地址”。

## 4. 不带头节点与带头节点

### 4.1 不带头节点

头指针直接指向第一个数据节点：

```text
head
  |
  v
+----+------+    +----+------+    +----+---------+
| 10 |   o--+--->| 20 |   o--+--->| 30 | nullptr |
+----+------+    +----+------+    +----+---------+
```

空表：

```cpp
head == nullptr
```

插入或删除第一个数据节点时，必须修改 `head` 本身，代码常需单独处理表头情况。

### 4.2 带头节点

头指针指向一个不计入线性表长度的辅助节点：

```text
head
  |
  v
+------+----+    +----+------+    +----+------+    +----+---------+
|  --  | o--+--->| 10 |   o--+--->| 20 |   o--+--->| 30 | nullptr |
+------+----+    +----+------+    +----+------+    +----+---------+
  头节点         第1个数据节点
```

空表仍保留头节点：

```cpp
head != nullptr
head->next == nullptr
```

头节点使“在第一个数据节点前插入”和“在其他节点前插入”都变成修改某个前驱节点的 `next`。

## 5. 头指针与头节点

```cpp
LNode *head;
```

两者含义不同：

```text
head       指针变量，保存头节点地址
*head      该地址处真正的头节点对象
head->next 头节点中保存的下一节点地址
```

```text
栈帧或全局区                    动态节点区

head = 0x5000  ----------------> 地址 0x5000 的头节点
                                 +------+----------+
                                 | data | next     |
                                 +------+----------+
```

## 6. 初始化带头节点的单链表

采用简单C++写法：

```cpp
bool initList(LNode *&head) {
    head = new LNode;
    head->next = nullptr;
    return true;
}
```

`LNode *&head` 是对调用者头指针的引用，因此函数内部给 `head` 赋值后，调用者也能得到新节点地址。

机器过程：

```text
1. 申请一个 LNode 对象
2. 将对象地址写入 head
3. 将头节点的 next 写为 nullptr
```

结果：

```text
head
  |
  v
+------+---------+
|  --  | nullptr |
+------+---------+
```

> 为突出数据结构操作，这里未展开内存分配失败与异常处理；工程代码还应根据使用的分配方式处理失败。

## 7. 按位定位节点

为了统一插入和删除，可以定义一个辅助函数：

```cpp
// 带头节点链表：
// position == 0 返回头节点；position >= 1 表示数据节点位序。
LNode *getNode(LNode *head, int position) {
    if (position < 0) {
        return nullptr;
    }

    LNode *p = head;
    int current = 0;

    while (p != nullptr && current < position) {
        p = p->next;
        ++current;
    }

    return p;
}
```

假设：

```text
头节点 -> 10 -> 20 -> 30
位序       1     2     3
```

调用 `getNode(head, 2)`：

```text
初始：p = 头节点，current = 0
第1次：p = 10，current = 1
第2次：p = 20，current = 2
停止并返回 p
```

若链表过短，`p` 会先变成 `nullptr`，函数返回空指针。

## 8. 按值查找

```cpp
LNode *locateElement(LNode *head, int target) {
    LNode *p = head->next; // 跳过头节点

    while (p != nullptr && p->data != target) {
        p = p->next;
    }

    return p; // 找到则返回节点地址，否则返回 nullptr
}
```

状态变化：

```text
p -> 10：10 == 30 ? 否
p -> 20：20 == 30 ? 否
p -> 30：30 == 30 ? 是，返回
```

最坏需要遍历全部 `n` 个数据节点，因此为 `O(n)`。

顺序表和链表的未排序按值查找都是 `O(n)`，但机器动作不同：

```text
顺序表：按连续地址逐项读取并比较
链表：  读取数据并额外读取 next，沿指针跳转
```

## 9. 单链表后插：改两条链接

已知 `p` 指向节点20，要在其后插入新节点99：

```text
插入前：20 -> 30
插入后：20 -> 99 -> 30
```

代码：

```cpp
bool insertAfter(LNode *p, int value) {
    if (p == nullptr) {
        return false;
    }

    LNode *s = new LNode;
    s->data = value;

    s->next = p->next;
    p->next = s;
    return true;
}
```

逐语句状态：

```text
初始：

p
|
v
+----+------+        +----+---------+
| 20 |   o--+------->| 30 | nullptr |
+----+------+        +----+---------+

执行 s->next = p->next：

p -> [20 | next] ------------> [30 | nullptr]
                                    ^
                                    |
s -> [99 | next] ------------------+

执行 p->next = s：

p -> [20 | next] -> [99 | next] -> [30 | nullptr]
```

已知 `p` 时，除节点分配外只修改固定数量的指针，408通常计为 `O(1)`。

## 10. 为什么不能随意交换插入语句

如果先执行：

```cpp
p->next = s;
```

且没有提前保存原来的 `p->next`，则状态变成：

```text
p -> 20 -> 99

原后继30的地址不再保存在 p->next 中
```

此时再执行：

```cpp
s->next = p->next;
```

由于 `p->next` 已经是 `s`，会得到：

```text
s->next == s
```

新节点形成自环，原后继链丢失。

链表修改的通用检查：

> 覆盖某个指针前，确认旧地址是否仍保存在其他变量或指针字段中。

## 11. 在第 i 个位置插入

要在第 `i` 个数据位置插入，必须修改第 `i - 1` 个节点的 `next`。带头节点时，第0个节点就是头节点：

```cpp
bool listInsert(LNode *head, int i, int value) {
    if (i < 1) {
        return false;
    }

    LNode *previous = getNode(head, i - 1);
    if (previous == nullptr) {
        return false;
    }

    return insertAfter(previous, value);
}
```

```text
原表：    头 -> 10 -> 20 -> 30
插入位序：       1     2     3     4

在第2个位置插入99：
找到第1个节点10
修改 10.next

结果：头 -> 10 -> 99 -> 20 -> 30
```

复杂度取决于位置是否已经找到：

```text
已知 previous：修改链接为 O(1)
只给位序 i：   先从头定位，整个操作最坏为 O(n)
```

## 12. 删除：让前驱跨过目标节点

原链：

```text
10 -> 20 -> 30
```

已知 `previous` 指向10，删除其后继20：

```cpp
bool deleteAfter(LNode *previous, int &removed) {
    if (previous == nullptr || previous->next == nullptr) {
        return false;
    }

    LNode *target = previous->next;
    removed = target->data;

    previous->next = target->next;
    delete target;
    return true;
}
```

逐语句状态：

```text
初始：
previous -> 10 -> 20 -> 30
                   ^
                   |
                 target

执行 previous->next = target->next：
previous -> 10 ----------> 30
                       
             20 -------> 30
             ^
             |
           target

执行 delete target：
释放原节点20，链表保留 10 -> 30
```

必须先用 `target` 保存待删节点地址。否则修改 `previous->next` 后，程序可能失去用于释放原节点的唯一地址，造成内存泄漏。

### 按位序删除

```cpp
bool listDelete(LNode *head, int i, int &removed) {
    if (i < 1) {
        return false;
    }

    LNode *previous = getNode(head, i - 1);
    return deleteAfter(previous, removed);
}
```

已知前驱时删除为 `O(1)`；只给位序时，定位前驱使整个操作最坏为 `O(n)`。

## 13. 为什么单链表不方便找前驱

已知 `p` 指向30：

```text
10 -> 20 -> 30 -> 40
            ^
            |
            p
```

节点30只保存数据和后继40的地址，没有前驱20的地址。只能从头查找满足以下条件的节点：

```cpp
candidate->next == p
```

最坏需要从头遍历，复杂度为 `O(n)`。这也是双链表增加 `prior` 指针的主要原因。

## 14. 头插法建立单链表

每次将新节点插到头节点之后：

```cpp
void createByHeadInsertion(LNode *head, const int values[], int n) {
    for (int i = 0; i < n; ++i) {
        LNode *s = new LNode;
        s->data = values[i];

        s->next = head->next;
        head->next = s;
    }
}
```

输入 `1, 2, 3, 4` 时：

```text
初始：头
读入1：头 -> 1
读入2：头 -> 2 -> 1
读入3：头 -> 3 -> 2 -> 1
读入4：头 -> 4 -> 3 -> 2 -> 1
```

结果与输入顺序相反。每次插入修改固定数量的指针，建立 `n` 个节点为 `O(n)`。

## 15. 尾插法为什么要维护尾指针

### 15.1 不维护尾指针

每插入一个节点都从头寻找表尾：

```text
第1次：寻找0步
第2次：寻找1步
第3次：寻找2步
...
第n次：寻找 n - 1 步
```

```text
总寻找次数
= 0 + 1 + 2 + ... + (n - 1)
= n * (n - 1) / 2

因此为 O(n^2)
```

### 15.2 维护尾指针

```cpp
void createByTailInsertion(LNode *head, const int values[], int n) {
    LNode *tail = head;

    for (int i = 0; i < n; ++i) {
        LNode *s = new LNode;
        s->data = values[i];
        s->next = nullptr;

        tail->next = s;
        tail = s;
    }
}
```

尾指针始终指向当前最后一个节点：

```text
读入1：头 -> 1
              ^
              tail

读入2：头 -> 1 -> 2
                   ^
                   tail

读入3：头 -> 1 -> 2 -> 3
                        ^
                        tail
```

每轮固定执行连接和更新尾指针，建立 `n` 个节点为 `O(n)`。

## 16. 顺序表与单链表对比

| 操作 | 顺序表 | 带头节点单链表 |
| --- | --- | --- |
| 按位访问 | 计算地址，`O(1)` | 沿 `next` 定位，`O(n)` |
| 未排序按值查找 | 逐项比较，`O(n)` | 沿链比较，`O(n)` |
| 已知节点后插 | 后缀可能需要搬移 | 修改两条链接，`O(1)` |
| 按位序插入 | 搬移后缀，`O(n)` | 定位前驱后修改链接，`O(n)` |
| 已知前驱删除其后继 | 后缀可能需要搬移 | 修改链接并释放，`O(1)` |
| 按位序删除 | 搬移后缀，`O(n)` | 定位前驱后删除，`O(n)` |
| 存储空间 | 连续区域 | 节点可分散，但每个节点还需指针字段 |

“链表插入、删除为 `O(1)`”必须附带前提：

```text
执行修改所需的相关节点已经找到
```

## 17. 双链表：同时保存前驱和后继

```cpp
struct DNode {
    int data;
    DNode *prior;
    DNode *next;
};
```

```text
一个双链表节点
+----------------+--------------+----------------+
| prior          | data         | next           |
| 前驱节点地址   | 当前数据     | 后继节点地址   |
+----------------+--------------+----------------+
```

逻辑结构：

```text
nullptr <- 10 <-> 20 <-> 30 -> nullptr
```

已知节点 `p` 时：

```cpp
p = p->next;  // 向后
p = p->prior; // 向前
```

## 18. 双链表后插

已知 `p` 指向A，在A之后插入S。先保存原后继：

```cpp
bool insertAfter(DNode *p, int value) {
    if (p == nullptr) {
        return false;
    }

    DNode *successor = p->next;
    DNode *s = new DNode;
    s->data = value;

    s->prior = p;
    s->next = successor;
    p->next = s;

    if (successor != nullptr) {
        successor->prior = s;
    }

    return true;
}
```

关系变化：

```text
插入前：A <-> B
插入后：A <-> S <-> B

S.prior = A
S.next  = B
A.next  = S
B.prior = S
```

若A原本是表尾，则 `successor == nullptr`，不能执行 `successor->prior`。这就是代码中条件判断的原因。

## 19. 双链表删除

已知 `target` 指向B：

```text
A <-> B <-> C
      ^
      target
```

```cpp
bool deleteNode(DNode *&head, DNode *target, int &removed) {
    if (target == nullptr) {
        return false;
    }

    DNode *previous = target->prior;
    DNode *successor = target->next;
    removed = target->data;

    if (previous != nullptr) {
        previous->next = successor;
    } else {
        head = successor; // target 原来是首节点
    }

    if (successor != nullptr) {
        successor->prior = previous;
    }

    delete target;
    return true;
}
```

普通非循环双链表的表头和表尾一侧可能是 `nullptr`，不能无条件解引用。若采用带哨兵的循环双链表，边界代码可以进一步统一。

## 20. 循环单链表

普通单链表表尾：

```text
last->next == nullptr
```

带头节点循环单链表表尾：

```text
last->next == head
```

```text
head -> 10 -> 20 -> 30
 ^                    |
 |____________________|
```

从第一个数据节点遍历时，结束条件不再是 `p == nullptr`：

```cpp
for (LNode *p = head->next; p != head; p = p->next) {
    // 访问 p->data
}
```

空表约定为：

```cpp
head->next == head
```

若只维护尾指针 `rear`：

```text
rear->next       是头节点
rear->next->next 是第一个数据节点（非空表时）
```

因此循环链表适合需要频繁在首尾之间切换或轮转访问的场景。

## 21. 指针修改的统一检查法

每次修改链接都按以下顺序检查：

```text
1. 当前有哪些指针指向旧节点？
2. 即将覆盖哪个 next 或 prior？
3. 被覆盖的旧地址是否已经保存？
4. 新节点的每个链接是否都有明确目标？
5. 前后节点是否也已经指回新节点？
6. 删除后是否仍有指针访问已释放节点？
7. 表头、表尾、空表和仅一个节点时是否仍成立？
```

## 22. 408易错点

1. 带头节点时，头节点不是第1个数据元素；可将它视作辅助的第0个节点。
2. 不带头节点空表为 `head == nullptr`；带头节点空表通常为 `head->next == nullptr`。
3. 循环链表不能用遇到 `nullptr` 作为一般遍历终点。
4. 单链表按位序插入和删除要先找前驱节点。
5. 覆盖 `next` 前必须确认原后继地址已经保存。
6. 删除节点后不能再解引用指向该节点的悬空指针。
7. 双链表非循环实现中，首节点的 `prior` 和尾节点的 `next` 可能为空。
8. 已知目标节点不等于已知它的前驱；单链表一般不能直接从目标节点找到前驱。
9. 链表节点地址通常是虚拟地址，不要把示意地址直接理解为物理内存地址。
10. C/C++结构体可能因对齐产生填充字节，节点实际大小不一定等于各字段表面大小简单相加。

## 最终机器模型

```text
单链表节点 = 数据 + 后继地址
双链表节点 = 前驱地址 + 数据 + 后继地址

查找：读取 next，一跳一跳移动
插入：先保存旧链接，再把新节点接入
删除：保存目标地址，让前驱跨过目标，再释放目标

顺序表靠计算地址找到元素；
链表靠读取节点中保存的地址找到后继。
```
