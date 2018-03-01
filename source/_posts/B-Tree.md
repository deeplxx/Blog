---
title: B-Tree
categories:
    - 算法
    - 数据结构
tag:
    - 数据结构
    - 大数据
mathjax: true
---

## B-树
1. B-树需要满足的条件：
    - 若不为空的情况下，根结点最少包含两个子结点
    - 所有的叶子结点都在同一层
    - 对于度为$m$的B树，每个结点最少包含$\lceil{m/2}\rceil$个子结点，最多包含$m$个子结点(度的定义有多种，理解意思就好，详见 [为什么 B-tree 在不同著作中度的定义有一定差别？](www.zhihu.com/question/19836260) 中的回答)
    - 对于包含t个关键值的结点，应当包含t+1个子结点 <!-- more -->
2. B-树高度：
    - 定义$t=\lceil{m/2}\rceil$即结点包含最少的子结点
    - 根结点(第一层)最少2个子结点，即第二层最少2个结点
    - 第$I$层最少有$2*t^{(I-2)}$个结点
    - 对于包含N个关键值的B树，最大高度为h, $$N=1+(t-1)\sum_{i=0}^{h-2}{2t^i}=1+2(t-1)\frac{t^{h-1}-1}{t-1}=2t^{h-1}-1$$其中$(t-1)$是每个结点包含的关键字个数。解得 $h=\log_t^{(n+1)/2}+1$
3. B-树的结构定义：
    ```c++
    template <typename T>
    struct BTree {
        int key_nums;
        BTree* parent;
        std::vector<BTree*> childs;
        std::vector<T> keys;
        BTree(int nums): key_nums(nums), childs(nums+1), keys(nums){}
    }
    ```
    ![B-树](http://p.blog.csdn.net/images/p_blog_csdn_net/manesking/4.JPG)
4. 特性：
    - 在文件索引中，一般将B树的结点大小定义为一个页的大小
    - 查找：B树的查询性能等价于二分查找
    - 插入：先查找待插入关键字的位置，若未满则直接插入，若满则需要分裂。分裂的方法：生成一新结点。把原结点上的关键字和k按升序排序后，从中间位置把关键字（不包括中间位置的关键字）分成两部分。左部分所含关键字放在旧结点中，右部分所含关键字放在新结点中，中间位置的关键字连同新结点的存储位置插入到父结点中。如果父结点的关键字个数也超过（m-1），则要再分裂，再往上插。直至这个过程传到根结点为止。
    ![B-树插入](http://img.blog.csdn.net/20170923211719582?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY3J5c3RhbDY5MTg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA/dissolve/70/gravity/SouthEast)
    - 删除：若删除后树结构定义，则需要合并。若要删除关键值所在结点的兄弟结点(左右)有多余的关键值，则将该兄弟结点的最大(最小)关键值上移到父节点，父节点的相应关键值下移到要删除关键值所在结点。若没有兄弟结点有多余关键值。假设该结点有左兄弟，且其左兄弟结点地址由其双亲结点指针Ai所指。则在删除关键字之后，它所在结点的剩余关键字和指针，加上双亲结点中的关键字Ki一起，合并到Ai所指兄弟结点中（若无左兄弟，则合并到右兄弟结点中）。如果因此使双亲结点中的关键字数目不满足条件，则依次类推。
    ![B-树删除](http://img.blog.csdn.net/20170923212229778?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY3J5c3RhbDY5MTg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA/dissolve/70/gravity/SouthEast)
    ![B-树删除](http://img.blog.csdn.net/20170923212525937?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY3J5c3RhbDY5MTg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA/dissolve/70/gravity/SouthEast)

## B+树
1. B+与B的异同：
    - B树的结构改善了磁盘IO效率，但没有解决元素**遍历**效率低下的问题，而在数据库中，根据范围查询的操作是非常普遍的
    - 结点的子树指针与关键值个数相同
    - 非叶子结点的子树指针P[i]，指向关键字值属于[K[i], K[i+1])的子树(注意左区间为闭，即子节点包含父节点相应的关键值)
    - 所有关键字都在叶子节点中出现
    - 所有叶子节点都有一个链指针
    - 非叶子结点相当于叶子节点的索引，叶子节点才是存储数据的数据层。(B+tree的内部结点并没有指向关键字具体信息的指针。即非叶子结点只存储数据的索引，不保存数据的所有信息)
    ![B+树](http://p.blog.csdn.net/images/p_blog_csdn_net/manesking/5.JPG)
2. 特性
    - 搜索：B+树只有到达叶子节点才命中

## B*树

1. 异同：
    - 在B+树的基础上，为非叶子节点也增加了链指针
    - 在B-树的基础上，每个结点关键字个数最少为$\lceil{2m/3}\rceil$。空间使用率更高
    ![B+树](http://hi.csdn.net/attachment/201106/7/8394323_13074405869mSW.jpg)
