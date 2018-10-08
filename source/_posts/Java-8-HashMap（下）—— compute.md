---
title: Java 8 HashMap(下)—— compute
date: 2018-09-24 00:19:04
categories:
- Java
tags:
- 公众号文章
- Java
- HashMap
---

{% asset_img IMG_0011.JPG 故乡的月 %}


*推荐阅读时间：5分钟*


### 简介
本篇接上篇 [Java 8 HashMap (上)—— 红黑树](https://jasonsonghoho.github.io/2018/09/24/Java-8-HashMap-%E4%B8%8A-%E2%80%94%E2%80%94-%E7%BA%A2%E9%BB%91%E6%A0%91/)，
继续探讨 Java8 的HashMap 的新特性。内容不多，重点介绍 compute 方法。


### compute

compute 方法主要用来将一个复杂计算的结果作为值赋给指定的键。key 指待修改或插入的键，remappingFunction 是一个 BiFunction 对象。
BiFunction 是指一个 二元（binary）函数；定义时，指定两个入参字段类型和一个结果字段类型。通过实现 apply 方法来自定义计算逻辑。

BiFunction 部分源码如下：
``` java
public interface BiFunction<T, U, R> {

    /**
     * Applies this function to the given arguments.
     *
     * @param t the first function argument
     * @param u the second function argument
     * @return the function result
     */
    R apply(T t, U u);
}
```


compute 方法源码如下：
``` java
public V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
    if (remappingFunction == null)
        throw new NullPointerException();
    int hash = hash(key);
    Node<K,V>[] tab; Node<K,V> first; int n, i;
    int binCount = 0;
    TreeNode<K,V> t = null;
    Node<K,V> old = null;
    if (size > threshold || (tab = table) == null ||
        (n = tab.length) == 0)
        n = (tab = resize()).length;
    //根据 key 值，找到对应的 旧值
    if ((first = tab[i = (n - 1) & hash]) != null) {
        if (first instanceof TreeNode)
            old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);
        else {
            Node<K,V> e = first; K k;
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) {
                    old = e;
                    break;
                }
                ++binCount;
            } while ((e = e.next) != null);
        }
    }
    V oldValue = (old == null) ? null : old.value;
    //计算出新值
    V v = remappingFunction.apply(key, oldValue);
    if (old != null) {
        if (v != null) {
            old.value = v;
            afterNodeAccess(old);
        }
        else
        //如果新值为 null，删掉这个 Node
            removeNode(hash, key, null, false, true);
    }
    //如果新值不为 null，替换这个 Node
    else if (v != null) {
        if (t != null)
            t.putTreeVal(this, tab, hash, key, v);
        else {
            tab[i] = newNode(hash, key, v, first);
            if (binCount >= TREEIFY_THRESHOLD - 1)
                treeifyBin(tab, hash);
        }
        ++modCount;
        ++size;
        afterNodeInsertion(true);
    }
    return v;
}

```

computeIfAbsent、computeIfPresent 方法与 compute 类似，分别表示当相应键的值缺席、存在时，才进行 compute 方法。
源码与 compute 基本一致，不赘述。

demo 如下：

``` java
public static void main(String[] args) {
    Map<String, String> hashMap = new LinkedHashMap<>();
    hashMap.put("k1", "v1");
    hashMap.put("k2", "v2");
    hashMap.put("k3", "v3");
    Function<String, String> function = new Function<String, String>() {
        @Override
        public String apply(String s) {
            return "test_" + s;
        }
    };
    BiFunction<String, String, String> biFunction = new BiFunction<String, String, String>() {
        @Override
        public String apply(String s, String s2) {
            return "test_" + s + "_" + s2;
        }
    };
    hashMap.compute("k1", biFunction);//生效
    hashMap.compute("k11", biFunction);//生效
    hashMap.computeIfAbsent("k2", function);//不生效
    hashMap.computeIfAbsent("k21", function);//生效
    hashMap.computeIfPresent("k3", biFunction);//生效
    hashMap.computeIfPresent("k31", biFunction);//不生效
    System.out.println(hashMap);
}

```

结果：
``` java
{k1=test_k1_v1, k2=v2, k3=test_k3_v3, k11=test_k11_null, k21=test_k21}
```


### merge

merge 方法是根据旧值进行计算，修改相应 Node。与 compute 结构相似，不过 BiFunction 入参为 oldValue,newValue。
源码与 compute 基本一致，不赘述。


### 其他

其他新增的方法如下：

``` java
@Override
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}

@Override
public V putIfAbsent(K key, V value) {
    return putVal(hash(key), key, value, true, true);
}

@Override
public boolean remove(Object key, Object value) {
    return removeNode(hash(key), key, value, true, true) != null;
}

@Override
public boolean replace(K key, V oldValue, V newValue) {
    Node<K,V> e; V v;
    if ((e = getNode(hash(key), key)) != null &&
        ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
        e.value = newValue;
        afterNodeAccess(e);
        return true;
    }
    return false;
}

@Override
public V replace(K key, V value) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) != null) {
        V oldValue = e.value;
        e.value = value;
        afterNodeAccess(e);
        return oldValue;
    }
    return null;
}

```

1. getOrDefault 方法根据 key 获取相应的值，如果为 null，则返回设置的默认值。
2. putIfAbsent 方法会在 指定的 key 不存在时，插入相应的值。
3. remove(Object key, Object value) 方法会根据 key 和 value 进行删除，如果 相应 key 的值不为该 value，则不执行删除并返回 false。
4. replace(K key, V oldValue, V newValue) 与 replace(K key, V value) 都是替换值的操作，具体区别与上述方法类似。



### 最后

HashMap历时一个半月，终于告一段落了😂。

由于在源码分析系列含有大量代码，放在公众号上不方便阅读，后续写的话会发在博客上。有兴趣的欢迎访问 http://jasonsonghoho.github.io 。


---


{% asset_img IMG_0007.JPG  %}



*“你不知
这风雪一程
有太多不易”*




