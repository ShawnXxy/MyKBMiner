# MySQL · 内核特性 · Btree 顺序插入优化及问题

**Date:** 2021/06
**Source:** http://mysql.taobao.org/monthly/2021/06/05/
**Images:** 9 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 06
 ](/monthly/2021/06)

 * 当期文章

 MySQL · 性能优化 · Undo Log IO优化
* MySQL · 源码分析 · Semi-join优化与执行逻辑
* MySQL · 源码分析 · Range (Min-Max Tree)结构分析
* MySQL · 源码分析 · Order By优化逻辑代码分析
* MySQL · 内核特性 · Btree 顺序插入优化及问题
* MySQL · 内核特性 · 分区表下的多种索引类型

 ## MySQL · 内核特性 · Btree 顺序插入优化及问题 
 Author: Yang Yuming 

 ## InnoDB 中的 B-tree

InnoDB 引擎使用索引组织表，即将所有数据记录有序存放在一个 B-tree 结构中，实现两个目的：

* 利用 B-tree 动态地组织磁盘文件结构，维护数据记录有序；
* 借助 B-tree 快速定位记录（B-tree 就是一个多级索引）；

InnoDB 实现的 B-tree 有几点特性：

* 数据记录全部存储在 leaf 层（即 B+ tree，降低树高度、优化顺序访问）；
* non-leaf 层节点中存储索引项（key, page no），每个索引项指向唯一一个 child 节点；
* 一个索引项的 key 为 P，它的 child 节点只能存 >= P 并且 < P1 的记录，其中 P1 是下一个索引项的 key；
* 每层节点通过双向链表串起来；

## B-tree 页面分裂

如果要在下图 B-tree （fan-out=4）上继续插入记录 9，InnoDB 首先以 <= 9 的条件会定位到 leaf 层的记录 8 上，但是发现该 page 已经没有更多空间，此时就需要申请一个 new page。

这里产生一个问题：如何选择分裂点（split point），把哪些 rec 移动到 new page？

![image-20210705104919981](.img/3fbab9781c8e_image-20210705104807777.png)

InnoDB 采用两种分裂策略：

### 中间点（mid point）分裂

将原始页面中 50% 数据移动到新页面，这是最普通的分裂方法。以上图为例，分裂后 5、6 保留在原页面，7、8 移动到新页面，并将 9 插入到 8 之后，调整树结构后如下图：

![image-20210705104942951](.img/eafa18594d85_image-20210705104942951.png)

这种分裂方法使两个 page 的空闲率相同，如果之后的插入在这两个 page 上是随机的，那可以很好地利用空闲空间。但是，如果后续插入不是随机的，比如递增插入 10、11、12 等等，填充和分裂的永远是右侧 page，左侧 page 的利用率只有 50%，如下图：

![image-20210705105006993](.img/f1c5b1530332_image-20210705105006993.png)

### 插入点（insert point）分裂

为了优化上述中间点分裂在顺序插入场景的问题，InnoDB 实现了在插入点分裂的方法，在每个 page 上记录上次插入位置 （PAGE_LAST_INSERT），以此判断本次插入是否递增 or 递减，如果判定为顺序插入，就在当前插入点进行分裂。还是以插入记录 9 为例，假设上次插入的是记录 8，本次插入时会判定为递增，在当前位置分裂后如下图：

![image-20210705105158872](.img/0070bddbafe1_image-20210705105158872.png)

此后，继续插入记录 10、11、12 都无需分裂，直到插入 13 时才会再次按插入点分裂一次：

![image-20210705105238631](.img/e4b246fe6b69_image-20210705105238631.png)

（注意，按插入点分裂并不一定发生在 page 的最后一个 rec，如果 PAGE_LAST_INSERT 在 page 中间，并且判定当前插入为顺序插入，也会在插入点进行分裂。）

### left split 优化

InnoDB 判断插入为递减模式时，会将 page 进行向左分裂，即 new page 插入到当前 page 左侧，这样做有两个优势：

* 递减插入模式时，只需要移动少量数据记录到左侧的 new page；
* 申请 new page 时，优先分配 page no 更小的页，这样的持续递减插入时，B-tree 从左到右的 page no 是保持递增的；

## Bug #67718

可以看出，在持续顺序插入情况下，B-tree 页面的空间利用率接近 100%。但是，在顺序插入和随机插入混合的情况下可能不起作用，甚至极端情况会导致极低的空间利用率（[Bug#67718](https://bugs.mysql.com/bug.php?id=67718)）：

假设 B-tree 中原本有记录 5、6、7、8，并且上次插入的是记录 8，我们依次插入 11、10、9，插入 11 时会判定为递增插入，按插入点分裂后，11 被单独放到右侧 new page 中：

![image-20210705105416171](.img/33c1530a9356_image-20210705105416171.png)

继续插入 10，按 InnoDB 遍历 B-tree 的方法会定位到记录 8，此时这个 page 的 PAGE_LAST_INSERT 还是 8，插入 10 又会被判定为递增插入！如果继续插入 9，还会定位到记录 8，最终导致 9、10、11 都独自占据一个 page，空间利用率极低：

![image-20210705105459939](.img/82a60c5d21a0_image-20210705105459939.png)

我们看到，问题在于每次都定位到记录 8（end of page），并且都判定为递增模式。

根本原因是：

* 如果要插入的 rec 在两个 page 的交界（gap）处，InnoDB 采用 <= 查找插入位置，会定位并插入到 left page 的最后，而不是 right page 的开头（因此上面 B-tree 连续插入 1、2、3 就没有这个问题）；
* 记录上一次插入的位置 PAGE_LAST_INSERT 只是 page 级别的，无法识别全局插入模式；

### 修复

针对这个问题，官方选择的修复方法是：如果插入点在 page 最后，就先尝试插入到其 next page 的开头，具体参考 `btr_insert_into_right_sibling` 函数。

将 rec 插入 right sibling 开头，会导致父节点中指向 right sibling 的索引项失效，例如，插入记录 10 后，node pointer 中的 key 还是 11，破坏了 B-tree 结构：

![image-20210705105526463](.img/77832f03e6bf_image-20210705105526463.png)

必须更新父节点的索引项的 key 为 10，如果此时父节点没有加锁，就需要申请父节点的 latch，导致持有下层 latch 去申请上层 latch 的情况，破坏了加锁顺序，从而导致 InnoDB 的 B-tree 并发控制无法采用 latch coupling。

这种方法增加了分裂的复杂度，也不能保证更极端的数据分布下没有问题。所以针对这个问题，也可以采用更简单直接的办法，例如只对 B-tree 中最左和最右 page 采用顺序插入的优化，其余 page 只使用中间点分裂；或者按插入点分裂时，将插入点反向移动一个 rec，即多移动一个 rec，对上面问题也是有效的。每个方法都有自己的适用场景，没有绝对的优劣势。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)