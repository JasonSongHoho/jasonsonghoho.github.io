---
title: Java 8 HashMap(上)—— 红黑树
date: 2018-09-24 00:19:04
categories:
- Java
tags:
- 公众号文章
- Java
- HashMap
- 红黑树
---



*推荐阅读时间：10分钟*


### 简介

>Java8的最大特性是使用红黑树结构来存储每个（桶）bucket中的数据（当链表长度超过8时）。  
为什么引入红黑树呢？其实正常情况下，平均每个桶中应该只会有不到1个数据，但当发生大量Hash碰撞时，每个桶中的数据也将会大量增加，这将会影响到数据的查询速度。在Java7中，每个桶使用链表存储数据，查找数据采用遍历的方式，查询时间复杂度为O(n)。Java7中为了避免大量Hash碰撞的问题，引入了hashseed方式，增强了Hash函数的散列性。但是randomHashSeed方法调用的next方法在多线程运算时存在性能问题（待考证），故在Java8中被弃用。Java8中换了一个思路：使用红黑树来提高查找速度（O(logN)），即使发生大量hash碰撞也不会造成性能影响，这便是红黑树由来的原因。  
Java7的HashMap一共有接近1200行代码，而到了Java8直接增加到了2400行，减去全局变量前新加的100行注释，相差1100行。其中TreeNode(红黑树)实现部分有600行代码，再加上其他方法对红黑树的适应性改动，可见红黑树部分是Java8 HashMap的主要改动。  



### 详情
#### 继承关系如下：

{% asset_img 20180923214733.png %}


TreeNode继承了LinkedHashMap.Entry，LinkedHashMap.Entry继承了HashMap.Node，而Node其实就是上个版本的Entry（链表）结构。此处有个疑惑：TreeNode并没有使用LinkedHashMap.Entry的before和after字段，不知道为啥不直接继承Node类。



#### 字段和构造函数如下：
``` java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    //常见的树结构
    HashMap.TreeNode<K, V> parent;
    HashMap.TreeNode<K, V> left;
    HashMap.TreeNode<K, V> right;
    HashMap.TreeNode<K, V> prev;
    boolean red;

    TreeNode(int hash, K key, V val, HashMap.Node<K, V> next) {
        super(hash, key, val, next);
    }
}

```

#### 红黑树全部方法如下：

``` java
/**
 * 获取根结点
 */ 
final HashMap.TreeNode<K,V> root() {
     for (HashMap.TreeNode<K,V> r = this, p;;) {
        if ((p = r.parent) == null)
            return r;
        r = p;
    }
}

/**
 * 选取根结点
 * 红黑树进行左旋、右旋后，根结点可能移动，头结点需要重新指向新的根结点
 */
static <K,V> void moveRootToFront(HashMap.Node<K,V>[] tab, HashMap.TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) {
        int index = (n - 1) & root.hash;
        HashMap.TreeNode<K,V> first = (HashMap.TreeNode<K,V>)tab[index];
        //如果根结点不是桶的第一个节点，则将根结点移动到头结点位置
        if (root != first) {
            HashMap.Node<K,V> rn;
            tab[index] = root;
            HashMap.TreeNode<K,V> rp = root.prev;
            if ((rn = root.next) != null)
                ((HashMap.TreeNode<K,V>)rn).prev = rp;
            if (rp != null)
                rp.next = rn;
            if (first != null)
                first.prev = root;
            root.next = first;
            root.prev = null;
        }
        //断言红黑树结构是否正常，不正常则报异常停止
        assert checkInvariants(root);
    }
}

/**
 * 根据结点hash值、key和key的类型 进行红黑树查找
 *
 */
final HashMap.TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    HashMap.TreeNode<K,V> p = this;
    do {
        int ph, dir; K pk;
        HashMap.TreeNode<K,V> pl = p.left, pr = p.right, q;
        //待查询结点的key的hash值若小于当前结点则进入左子树，否则进入右子树
        if ((ph = p.hash) > h)
            p = pl;
        else if (ph < h)
            p = pr;
        //如果hash值相等再比较key，一致则表示找到，返回结果
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if (pl == null)
            p = pr;
        else if (pr == null)
            p = pl;
        //如果key不同，则看key的类型是否可比较，
        //如果可以比较则根据比较结果选择是否返回查询结果，还是继续查询左、右子树
        else if ((kc != null ||
                (kc = comparableClassFor(k)) != null) &&
                (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr;
        else if ((q = pr.find(h, k, kc)) != null)
            return q;
        else
            p = pl;
    } while (p != null);
    return null;
}

/**
 * 根据结点hash值、key 进行红黑树查找
 */
final HashMap.TreeNode<K,V> getTreeNode(int h, Object k) {
    return ((parent != null) ? root() : this).find(h, k, null);
}

/**
 * 强制比较两个结点的key
 *
 * 当两个类型不能直接比较时，通过对类名进行hash进行比较；若hash值也相等，判定b大。
 * 故一定会排出先后顺序。
 */
static int tieBreakOrder(Object a, Object b) {
    int d;
    if (a == null || b == null ||
            (d = a.getClass().getName().
                    compareTo(b.getClass().getName())) == 0)
        d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                -1 : 1);
    return d;
}

/**
 * 将链表转化为红黑树
 * 重要方法！当链表长度增加到红黑树转换阈值（默认8），且桶的数量不小于64 时触发
 */
final void treeify(HashMap.Node<K,V>[] tab) {
    HashMap.TreeNode<K,V> root = null;
    //遍历链表
    for (HashMap.TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (HashMap.TreeNode<K,V>)x.next;
        x.left = x.right = null;
        //首结点作为根结点
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (HashMap.TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                        (kc = comparableClassFor(k)) == null) ||
                        (dir = compareComparables(kc, k, pk)) == 0)
                    //如果未能比较，调用强制比较方法，确保有序
                    dir = tieBreakOrder(k, pk);

                HashMap.TreeNode<K,V> xp = p;
                //当左或右子树为空时，插入链表上的一个结点
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    //插入结点后，进行左旋、右旋重平衡，转换为红黑树
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    //确保根结点是首结点
    moveRootToFront(tab, root);
}

/**
 * 红黑树转换为链表
 * 当红黑树结点减少到链表还原阈值（默认6）时触发
 */
final HashMap.Node<K,V> untreeify(HashMap<K,V> map) {
    HashMap.Node<K,V> hd = null, tl = null;
    for (HashMap.Node<K,V> q = this; q != null; q = q.next) {
        HashMap.Node<K,V> p = map.replacementNode(q, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}

/**
 * 红黑树插入新结点
 */
final HashMap.TreeNode<K,V> putTreeVal(HashMap<K,V> map, HashMap.Node<K,V>[] tab,
                                       int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    HashMap.TreeNode<K,V> root = (parent != null) ? root() : this;
    for (HashMap.TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if ((kc == null &&
                (kc = comparableClassFor(k)) == null) ||
                (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                HashMap.TreeNode<K,V> q, ch;
                searched = true;
                //插入结点与当前结点hash值相等时，查找左子树或右子树，若已包含待插入结点则直接返回结果
                if (((ch = p.left) != null &&
                        (q = ch.find(h, k, kc)) != null) ||
                        ((ch = p.right) != null &&
                                (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        }

        HashMap.TreeNode<K,V> xp = p;
        //若已比较到左子树或右子树为空时还没有找到，则插入该结点
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            HashMap.Node<K,V> xpn = xp.next;
            HashMap.TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((HashMap.TreeNode<K,V>)xpn).prev = x;
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}

/**
 * 删除红黑树结点
 * 呃，这个方法和前面的比起来大同小异，不细看了
 */
final void removeTreeNode(HashMap<K,V> map, HashMap.Node<K,V>[] tab,
                          boolean movable) {
    int n;
    if (tab == null || (n = tab.length) == 0)
        return;
    int index = (n - 1) & hash;
    HashMap.TreeNode<K,V> first = (HashMap.TreeNode<K,V>)tab[index], root = first, rl;
    HashMap.TreeNode<K,V> succ = (HashMap.TreeNode<K,V>)next, pred = prev;
    if (pred == null)
        tab[index] = first = succ;
    else
        pred.next = succ;
    if (succ != null)
        succ.prev = pred;
    if (first == null)
        return;
    if (root.parent != null)
        root = root.root();
    if (root == null || root.right == null ||
            (rl = root.left) == null || rl.left == null) {
        //若删除结点后，达到链表还原阈值，则还原为链表
        tab[index] = first.untreeify(map);
        return;
    }
    HashMap.TreeNode<K,V> p = this, pl = left, pr = right, replacement;
    if (pl != null && pr != null) {
        HashMap.TreeNode<K,V> s = pr, sl;
        while ((sl = s.left) != null) // find successor
            s = sl;
        boolean c = s.red; s.red = p.red; p.red = c; // swap colors
        HashMap.TreeNode<K,V> sr = s.right;
        HashMap.TreeNode<K,V> pp = p.parent;
        if (s == pr) { // p was s's direct parent
            p.parent = s;
            s.right = p;
        }
        else {
            HashMap.TreeNode<K,V> sp = s.parent;
            if ((p.parent = sp) != null) {
                if (s == sp.left)
                    sp.left = p;
                else
                    sp.right = p;
            }
            if ((s.right = pr) != null)
                pr.parent = s;
        }
        p.left = null;
        if ((p.right = sr) != null)
            sr.parent = p;
        if ((s.left = pl) != null)
            pl.parent = s;
        if ((s.parent = pp) == null)
            root = s;
        else if (p == pp.left)
            pp.left = s;
        else
            pp.right = s;
        if (sr != null)
            replacement = sr;
        else
            replacement = p;
    }
    else if (pl != null)
        replacement = pl;
    else if (pr != null)
        replacement = pr;
    else
        replacement = p;
    if (replacement != p) {
        HashMap.TreeNode<K,V> pp = replacement.parent = p.parent;
        if (pp == null)
            root = replacement;
        else if (p == pp.left)
            pp.left = replacement;
        else
            pp.right = replacement;
        p.left = p.right = p.parent = null;
    }

    HashMap.TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

    if (replacement == p) {  // detach
        HashMap.TreeNode<K,V> pp = p.parent;
        p.parent = null;
        if (pp != null) {
            if (p == pp.left)
                pp.left = null;
            else if (p == pp.right)
                pp.right = null;
        }
    }
    if (movable)
        moveRootToFront(tab, r);
}

/**
 * 将红黑树分为两半
 * 扩容时，会将一个桶中的红黑树拆分为两个；若拆分后红黑树不够大，会被还原为链表。
 */
final void split(HashMap<K,V> map, HashMap.Node<K,V>[] tab, int index, int bit) {
    HashMap.TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    HashMap.TreeNode<K,V> loHead = null, loTail = null;
    HashMap.TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (HashMap.TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (HashMap.TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    if (loHead != null) {
        if (lc <= UNTREEIFY_THRESHOLD)
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}

/* ------------------------------------------------------------ */
// Red-black tree methods, all adapted from CLR

/**
 * 红黑树的左旋
 * 红黑树相关知识待深入学习后进一步探讨
 */
static <K,V> HashMap.TreeNode<K,V> rotateLeft(HashMap.TreeNode<K,V> root,
                                              HashMap.TreeNode<K,V> p) {
    HashMap.TreeNode<K,V> r, pp, rl;
    if (p != null && (r = p.right) != null) {
        if ((rl = p.right = r.left) != null)
            rl.parent = p;
        if ((pp = r.parent = p.parent) == null)
            (root = r).red = false;
        else if (pp.left == p)
            pp.left = r;
        else
            pp.right = r;
        r.left = p;
        p.parent = r;
    }
    return root;
}

/**
 * 红黑树的右旋
 */
static <K,V> HashMap.TreeNode<K,V> rotateRight(HashMap.TreeNode<K,V> root,
                                               HashMap.TreeNode<K,V> p) {
    HashMap.TreeNode<K,V> l, pp, lr;
    if (p != null && (l = p.left) != null) {
        if ((lr = p.left = l.right) != null)
            lr.parent = p;
        if ((pp = l.parent = p.parent) == null)
            (root = l).red = false;
        else if (pp.right == p)
            pp.right = l;
        else
            pp.left = l;
        l.right = p;
        p.parent = l;
    }
    return root;
}

/**
 * 插入结点后进行重平衡
 * 这部分只看懂代码了，算法比较懵，有机会再探讨红黑树算法
 */
static <K,V> HashMap.TreeNode<K,V> balanceInsertion(HashMap.TreeNode<K,V> root, HashMap.TreeNode<K,V> x) {
    //将插入的结点设为红色
    x.red = true;
    for (HashMap.TreeNode<K,V> xp, xpp, xppl, xppr;;) {
        //x为根结点时，设为黑色，直接返回
        if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        //父结点为黑且为根结点时，直接返回。
        else if (!xp.red || (xpp = xp.parent) == null)
            return root;
        //父结点为祖父结点的左孩子结点时：
        if (xp == (xppl = xpp.left)) {
            //祖父结点的右孩子结点为红色时，叔叔结点和父结点置黑，祖父结点置红，设置当前结点为祖父结点
            if ((xppr = xpp.right) != null && xppr.red) {
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                //否则，如果当前结点为父结点的右孩子结点，进行左旋
                if (x == xp.right) {
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                //如果父结点不为空设为黑色，且如果祖父结点不为空则设为红色并进行右旋
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }
        else {
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                if (x == xp.left) {
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
/**
 * 删除结点后进行重平衡
 */
static <K,V> HashMap.TreeNode<K,V> balanceDeletion(HashMap.TreeNode<K,V> root,
                                                   HashMap.TreeNode<K,V> x) {
    for (HashMap.TreeNode<K,V> xp, xpl, xpr;;)  {
        if (x == null || x == root)
            return root;
        else if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        else if (x.red) {
            x.red = false;
            return root;
        }
        else if ((xpl = xp.left) == x) {
            if ((xpr = xp.right) != null && xpr.red) {
                xpr.red = false;
                xp.red = true;
                root = rotateLeft(root, xp);
                xpr = (xp = x.parent) == null ? null : xp.right;
            }
            if (xpr == null)
                x = xp;
            else {
                HashMap.TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                if ((sr == null || !sr.red) &&
                        (sl == null || !sl.red)) {
                    xpr.red = true;
                    x = xp;
                }
                else {
                    if (sr == null || !sr.red) {
                        if (sl != null)
                            sl.red = false;
                        xpr.red = true;
                        root = rotateRight(root, xpr);
                        xpr = (xp = x.parent) == null ?
                                null : xp.right;
                    }
                    if (xpr != null) {
                        xpr.red = (xp == null) ? false : xp.red;
                        if ((sr = xpr.right) != null)
                            sr.red = false;
                    }
                    if (xp != null) {
                        xp.red = false;
                        root = rotateLeft(root, xp);
                    }
                    x = root;
                }
            }
        }
        else { // symmetric
            if (xpl != null && xpl.red) {
                xpl.red = false;
                xp.red = true;
                root = rotateRight(root, xp);
                xpl = (xp = x.parent) == null ? null : xp.left;
            }
            if (xpl == null)
                x = xp;
            else {
                HashMap.TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                if ((sl == null || !sl.red) &&
                        (sr == null || !sr.red)) {
                    xpl.red = true;
                    x = xp;
                }
                else {
                    if (sl == null || !sl.red) {
                        if (sr != null)
                            sr.red = false;
                        xpl.red = true;
                        root = rotateLeft(root, xpl);
                        xpl = (xp = x.parent) == null ?
                                null : xp.left;
                    }
                    if (xpl != null) {
                        xpl.red = (xp == null) ? false : xp.red;
                        if ((sl = xpl.left) != null)
                            sl.red = false;
                    }
                    if (xp != null) {
                        xp.red = false;
                        root = rotateRight(root, xp);
                    }
                    x = root;
                }
            }
        }
    }
}

/**
 * 检查红黑树结构是否正常
 */
static <K,V> boolean checkInvariants(HashMap.TreeNode<K,V> t) {
    HashMap.TreeNode<K,V> tp = t.parent, tl = t.left, tr = t.right,
            tb = t.prev, tn = (HashMap.TreeNode<K,V>)t.next;
    if (tb != null && tb.next != t)
        return false;
    if (tn != null && tn.prev != t)
        return false;
    if (tp != null && t != tp.left && t != tp.right)
        return false;
    if (tl != null && (tl.parent != t || tl.hash > t.hash))
        return false;
    if (tr != null && (tr.parent != t || tr.hash < t.hash))
        return false;
    if (t.red && tl != null && tl.red && tr != null && tr.red)
        return false;
    if (tl != null && !checkInvariants(tl))
        return false;
    if (tr != null && !checkInvariants(tr))
        return false;
    return true;
}

```
{% asset_img IMG_1756.GIF %}
Java8的HashMap解读暂告一段落，下期继续探讨其他特性，希望不要太久😂


https://y.qq.com/n/yqq/song/004dbfuf1jEjpI.html#stat=y_new.index.new_song.songname




