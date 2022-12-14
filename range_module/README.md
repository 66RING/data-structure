# 线段树

- https://leetcode.cn/problems/range-module/

- 线段树的应用场景
    * 对区间的操作: 使用二分的方法快速找到目标区间
    * 题解是累加可解的, 即最终结果由左部和右部得到

本质上也是一种二叉搜索树, 只不过搜索的依据是区间范围。

首先是二叉搜索树, 搜索的是一个目标区间`[l, r]`, 然后我们有一个当前区间`[start, end]`

如果目标区间覆盖了当前区间`l <= start && end <= r`, 那就可以说找到了目标区间。

所以查找操作伪代码: 

```
find(当前区间, 目标区间) {
    if 目标区间 in 当前区间 {
        return 找到
    }
    let mid = 当前区间.中间值();

    // 分别用左右区间补齐
    if 目标区间.left() < mid {
        find(当前区间[:mid], 目标区间);
    }
    if 目标区间.right() >= mid {
        find(当前区间[右], 目标区间);
    }
}
```












