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

## 快进式合并
  如果当前的分支和另一个分支没有内容上的差异，就是说当前分支的每一个提交(commit)都已经存在另一个分支里了，git 就会执行一个“快速向前”(fast forward)操作；git 不创建任何新的提交(commit),只是将当前分支指向合并进来的分支。

  最直观的例子，从master上创建分支dev，然后在dev上提交代码，再将dev合并到master，这时候dev包含master所有的提交，git会直接将master指向dev，这就是快进式提交。

  是否在Merge执行快进式提交，主要看当前的分支最后一次提交的hash是否包含在需要合并的分支。

## 快进式合并的四个步骤
  1. HEAD分支(当前分支)指向master分支

  ![1]

  2. 从master分支创建dev分支，并将HEAD指向dev分支

  ![2]

  3. 在dev分支上提交代码

  ![3]

  4. 合并dev分支到master分支

  ![4]

## 合并
 * git merge -quiet <branch> 无声的合并（不会输出任何信息）
 * git merge -ff <branch> 当合并是快进式合并的时候，仅仅是更新了分支的指针，不会产生合并提交，这也是默认的合并行为
 * git merge -no-ff <branch> 即使是快进式合并，也会创建一个合并提交
 * git merge -ff-only <branch> 只允许快进式合并

  -ff快进式合并

  ![5]

   -no-ff非快进式合并

  ![6]

## git rebase 和 git merge

 rebase会把你当前分支的 commit 放到公共分支的最后面,所以叫变基。就好像你从公共分支又重新拉出来这个分支一样。

 例子：如果你从 master 拉了个feature分支出来,然后你提交了几个 commit,这个时候刚好有人把他开发的东西合并到 master 了,这个时候 master 就比你拉分支的时候多了几个 commit,如果这个时候你 rebase master 的话，就会把你当前的几个 commit，放到那个人 commit 的后面。

 ![7]

 merge 会把公共分支和你当前的commit 合并在一起，形成一个新的 commit 提交

 ![8]

 * 不要在公共分支使用rebase
 * 本地和远端对应同一条分支,优先使用rebase,而不是merge

## squash
如果不想在合并分支时体现多次commit记录, 可以增加squash参数
* git merge --squash feature-1.0.0
* git rebase -i dev
  ```
    pick 0483730 bar.txt 2222
    pick adf4d92 bar.txt 3333
    pick cd1b421 bar.txt 4444
  ```
  * 修改 pick 为 s
  ```
    pick 0483730 bar.txt 2222
    s adf4d92 bar.txt 3333
    s cd1b421 bar.txt 4444
  ```
  * 提交





[1]:img/1.png
[2]:img/2.png
[3]:img/3.png
[4]:img/4.png
[5]:img/5.jpg
[6]:img/6.jpg
[7]:img/7.webp
[8]:img/8.webp



