## 回溯框架
```
res = []
def backtrack(路径， 选择列表):
    if 满足条件:
        res.append(路径)
        return
    for 选择 in 选择列表:
        if 有效选择:
            路径.add(选择)
            选择列表.remove(选择)
            backtrack(新路径, 新选择列表)
            路径.remove(选择)
            选择列表.add(选择)
```

## 全排列

## N皇后

