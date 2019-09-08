# Git高级用法

## cherry-pick
cherry-pick可用于把其他分支的commit，移到当前分支。


## 创建干净历史的分支
```shell script
git checkout --orphan <new-branch>
```
--orphan 创建出来的分支没有任何提交历史

## git clone深度
```shell script
// 克隆仓库, 且只克隆最近 1 次提交历史, 如果想克隆最近 n 次的历史, 把 1 改成 n 即可
git clone --depth=1 <repository url>
```

## 查看某个文件的历史
```shell script
git log --all --full-history package-lock.json
```
* --all 展示包括其他分支的所有历史
* --full-history 关闭History Simplification
  * 查看某个文件历史时，git会自动简化历史（History Simplification），甚至会隐藏一些提交

