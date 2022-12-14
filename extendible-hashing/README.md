# 可扩展哈希

- https://github.com/nitish6174/extendible-hashing

- abs
    * bucket
    * directory

整体流程:

1. key根据全局深度找到桶号
2. 根据桶号找到对应的桶(哈希表), 做哈希表的插入
3. 桶满时要做分裂操作, 分裂包含两种类型
    1. 重新映射: 有的桶不需要分裂, 但因为目录扩大了, 要想下次哈希找到一样的桶, 就需要修改目录中桶的映射
    2. 重新分配: 桶本身元素已满, 需要分裂和元素迁移
4. 删除与合并


难点是如何分裂

TODO: 性能太拉了, 开销在split里的数据拷贝, 桶开大了(e.g. 1000)性能就行了


## rust tips

- trait
    * 哈希操作: 创建一个哈希器`DefaultHasher::new()`, 用它做哈希然后调用`finish`返回哈希结果
    * 为何RefCell不好: ⭐ review 理清, 理解
        + https://rust-unofficial.github.io/too-many-lists/fourth-peek.html
        + https://users.rust-lang.org/t/how-to-return-reference-to-value-in-rc-or-refcell/76729
- API
    * 使用`vec::reserve(additional)`预留空间, 防止频繁realloc
        + e.g. 创建新桶时, 目录扩展时
- 习俗
    * getter, setter命名规范: getter函数同名, setter函数前加个`set_`


### why not refcell

> Most operations will only touch the head or tail pointer. However when transitioning to or from the empty list, we need to edit both at once.

一旦使用`RefCell`就不能离开它(e.g. move out of), 因为它需要一套机制来维护运行时借用检测。所以如果你只能将RefCell中内容用Ref的API:`Ref::map()`移到另一个`Ref<T>`中。**但是我们想要原汁原味的`&T`**

RefCell会使用`Ref`/`RefMut`之类的东西包裹数据, 就是为了在`Ref`离开作用域后自动调用drop, 然后动态维护借用规则。


## bucket

⭐ 核心是找到桶号: `n&((1<<global_depth)-1)`

随着全局深度加深, 高位就会逐渐启用, 从而实现逐渐extend的过程

- `insert()`
    * 每个桶的本质上就是一个哈希表, 所以对桶的插入其实就是对哈希表的插入
    * 只不过我们还会限制大小, 然后做分裂
- `remove()`
    * 桶(哈希表)元素删除
    * 空桶多时做分裂
- `update()`
    * 简单的哈希元素更新
- `search()`
    * 简单的哈希查找
- `local_depth`
- ⭐`split()`
- `size`
    * 每个桶的分裂阈值

## directory

哈希桶的目录, 根据全局深度做哈希得到对应桶

- `insert()`
    * 根据全局深度`global_depth`找桶(一个桶就是一个哈希表)
    * 在桶上做哈希插入
    * 当桶大小超过阈值时桶分裂
    * 如果分裂的桶的深度`local_depth`与全局深度`global_depth`相同, 则目录需要扩容
- `remove()`
    * 根据全局深度找桶
    * 桶(哈希表)元素删除
    * 当桶空时, **针对桶号** 找机会合并
    * 三种模式: 仅删除(lazy), 删除加桶合并, 删除加桶合并加目录收缩
- `update()`
    * 根据全局深度找桶
    * 简单的哈希元素更新
- `search()`
    * 根据全局深度找桶
    * 简单的哈希查找
- ⭐`split()`
    * 桶重新分配(重映射 + 数据迁移)
    * 如果桶的深度和目录全局深度相同, 则目录需要扩展, 在重新分配
    * 否则就仅是当前桶数据的重新分配
    * 插入新entry然后重新分配
        1. 扩容: dep + 基本remap
        2. 迁移: 两种情况⭐`local_depth == global_depth`的桶和`local_depth != global_depth`的桶, 甚至可能`local_depth`落后`global_depth`很多
            - 大总结: 为什么需要(`local_depth`落后`global_depth`很多⭐⭐⭐)
                * 开新章节说明这种情况
                * ⭐⭐这种情况下即要分裂又要重新分配: 比如原来的桶被4个entry引用, 但是这个桶分裂了。应该会出现两个桶, 每个桶被2个entry引用。而不应该分裂后出现`1:3`的引用
- ⭐ `merge()`
    * 当前桶元素已被删除, 找与之对应的桶
    * 如果对应桶的深度和当前桶深度匹配则可以可并:
        + 删除当前桶
        + 对应桶领养当前的目录项
    * 当不存在`local_depth == global_depth`的桶时, 必然存在一个桶被前后两部分引用, 因此可以安全删除后半部分
- `shrink()`
    * 遍历所有桶, 当没有桶的深度等于全局深度时, 目录收缩
        + `global_depth--`
        + 释放目录中的桶(减半)
- `grow()`
    * 目录扩展
        + `global_depth++`
        + 增加目录中的桶(翻倍)
        + **相同后缀直接拷贝**. e.g. `0010`, `1010`相同后缀
            + 只需要**翻倍的同时顺序插入**即可: `for i in len(dir): dir.push(dir.buckets[i])`
- `hash()`
    * 哈希找到桶号
- `global_depth`
    * 当前目录大小


## 魔鬼细节

- mask的时候是加一还是减一
- map如何拷贝
    * 构造函数 + 迭代器
- double free bug
    * 桶id计算方式错误, 应该为`n&((1<<global_depth)-1)`
- 分裂后要根据新的depth做哈希找桶插入
- 需要注意已存在的kv就不做insert, **也不做split**
- 初始桶深度和全局深度相同
    * 初始状态必须是1:1映射的, 且桶深度应该和全局深度一致
        + 否则还得考虑哪些桶N:1, 他们的本地深度如何
- 注意内存预分配


### ⭐⭐⭐搞清楚几个关键的位运算

1. 根据全局深度查桶: `(1<<depth)-1`
    - 全局深度表示的是用多少bit数据做桶编号(aka 哈希范围)
    - 又因为位运算0 base: `1<<0`表示1, 所以做掩码的时候直接`(1<<depth) - 1`就可以得到低位截取值
2. 翻倍/减半
    - 深度表示用几bit数据, 容量c与位数n的关系是`c = 1<<n`, 当然依旧是0 base
3. ⭐找对应桶(组)

本质: 只关注后缀, 后缀是哈希依据。

对于一个桶只有两个目录项引用的情况, 如: 1号桶有`00`和`10`引用。

```
00 -> 1
01 -> 2
10 -> 1
11 -> 3
```

易知1号桶本地深度为1, 当1号桶分裂时, 数据将分摊到00和10项中。这是因为分裂后本地深度增加, 掩码位数增加, 使用2bit做哈希, 00, 10有别(仅1bit, 且是第local_depth bit, 不同), 所以数据重新分布。

⭐ 当一个桶有多于两个目录项引用的时候, 如:

```
000 -> 1
001
010 -> 1
011
100 -> 1
101
110 -> 1
111
```

这时1号桶的本地深度仍然是1, 因为其他桶的分裂导致1号桶会有很多目录向引用。这时1号桶分裂时该如何进行? 当只有1位后缀时4个目录项指向一个桶, 当深度升级到2bit时, 我们需要在全局3bit的范围内找到这4个中后缀相同的两组进行分配。

为什么必是两组: 因为本地深度每次分裂只增加1bit, 也就是说相同后缀范围内只有1bit不同, 所以是两组。

算法: 

1. 找到相同后缀范围内的另一组对应的后缀: `idx = bucket_num ^ (1<<old_local_depth)`. 即不同位将出现在最高位
2. 找到全局范围内是1中这个后缀的项, 归到一组: `for idx += (1<<new_local_depth)`

⭐或者重新分配可以使用这个算法: 即利用相同后缀计算。

```rust
for prefix in 0..(1 << (self.depth - local_depth - 1)) {
    let i = (idx & bitmask(local_depth) as usize)
        | 1 << local_depth
        | (prefix << (local_depth + 1)) as usize;
    self.table[i] = b1.clone();
}
```




