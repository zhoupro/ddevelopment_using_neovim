# skiplist

## 随机算法

ZKIPLIST_P 为 0.25。random 函数返回 0 到 2^32 - 1 之间的随机整数。
| 层数 | 概率                          |
|------|-------------------------------|
| 1    | (1 - 0.25)                    |
| 2    | 0.25*(1-0.25)                 |
| 3    | 0.25 * 0.25 * (1-0.25)        |
| 4    | 0.25 * 0.25 * 0.25 * (1-0.25) |
| n    | 0.25 ^(n-1) * (1-0.25)        |

```c
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```
