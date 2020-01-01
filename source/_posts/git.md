---
title: Git
categories:
  - 效率工具
  - Git
tags:
  - Git
date: 2019-06-18 23:53:16
---

# 合并多个commit

 <!-- more --> 

## 在push之前合并

情景：查看master分支的commit历史信息，会看到不同feature分支合并进来的许多commit。这么多粒度较小的commit，我们怎么知道哪些commit属于同一个feature。如果在master分支上，一个feature一个commit，这样会清晰很多，feature对外屏蔽掉开发过程中各种细小的commit，对外就体现出一个大commit。

假设目前的commit如下图所示，我们希望将这三个commit合并成一个commit（代码不改变）。例如我们在向开源社区提交一个较大的PR时，不可避免地会出现许多修修补补的commit，这些commit如果提交到master上面会非常难看且没有重点。因此，我们需要将开发过程中的多个commit合并成一个有明确含义的commit提交给reviewer。

```bash
commit b2acc4d4aa5ca9bb21c0a62cd36de1911a3a2931 (HEAD -> develop)
Date:   Thu Jun 13 09:35:48 2019 +0800

    develop33

commit 329f9878306ed7f1d37638f9a32564f393347811
Date:   Thu Jun 13 09:35:35 2019 +0800

    develop22

commit d6a56b89d4f6c7312837625766d6679aefe59816
Date:   Thu Jun 13 09:20:46 2019 +0800

    develop11
```

下面我们使用`git rebase -i HEAD~3`将最近的3个commit合并成一个，`-i`表明这是一个交互的操作，接下去需要我们去指示要如何操作这3个commit。一般场景下，推荐使用以下操作：将最早的commit（最上方）标识为r，代表修改commit信息，将其他较新的commit标识为fixup，代表丢弃commit信息，且代码合并进之前的commit。

```
r develop11  d6a56
fixup develop22  329f9
fixup develop33  b2acc
```

合并完成后会生成一个新的commit，这个commit包含了以上3个commit的所有改动，且会生成一个全新的hash，不同于以上被合并的3个commit。



## 在push之后合并

情景：在提交PR之后，同事进行Code Review，给出了一些修改意见。我在本地修改后，会多出一个commit。使用上述方法可以将其合并，这时使用`git push`，会提示错误。提示本地分支落后于远程分支，这时我们可以使用`git push -f`强制更新远程分支。



# 合并分支

这里我们可以拿git merge和git rebase来做对比。

```
M1--M2--M3(master)
     \
      \
       D1(develop)
```

## git merge

这个时候如果在develop分支上使用`git merge master`，则会将master分支合并至develop分支，且会保留原记录，形成非线性的commit记录。这种方式不太推荐。

```
M1--M2----M3--D2(develop)
        \    /
         \  /
          D1
```

## git rebase

`git rebase master`，会将master分支合并进develop分支，且记录是线性的，新生成的D1'这个commit的hash不同于D1。可以理解为D1'这个分支是基于M3进行修改的，而不是基于M2。

```
M1--M2--M3--D1'(develop)
```

# 拉取新分支

```
git pull = git fetch + git merge FETCH_HEAD 
git pull --rebase =  git fetch + git rebase FETCH_HEAD 
```

