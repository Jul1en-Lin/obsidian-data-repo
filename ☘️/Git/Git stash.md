# Git stash
> 相关笔记：[[Git|Git 知识总结]]


在使用 Git 开发时，我们经常会遇到一种很尴尬的情况：当前功能还没写完，代码处于“半成品”状态，既不适合提交 commit，也不想直接丢掉。但这时突然来了一个新的任务，比如需要马上切换分支去修 Bug，或者需要拉取远程最新代码。

问题就出现了：Git 默认不太喜欢你带着一堆未提交的修改到处切换分支。因为这些修改可能会和目标分支的文件产生冲突，Git 无法确定你到底想保留哪一份内容。

这时候就需要 `git stash`。

​`stash` 的作用可以理解为：把当前还不想提交的修改临时存起来，让工作区恢复到干净状态。等你处理完别的事情后，再把这些修改取回来继续写。

## 核心理解

Git 里常见的代码状态主要有三个区域：

|区域|含义|
| -----------------------------| ------------------------|
|工作区 Working Directory|你正在编辑的文件内容|
|暂存区 Staging Area / Index|已经`git add`，准备提交的内容|
|本地仓库 Repository|已经 commit 的历史版本|

普通的 `git commit` 是把暂存区内容正式保存到版本历史里。

而 `git stash` 是把当前工作区和暂存区里的修改临时打包，放进一个 Git 管理的临时栈里。

> 当前写到一半的代码 -> git stash -> 临时保存到 stash 栈中 -> 工作区恢复干净 -> 切换分支 / 拉代码 / 修 Bug -> git stash pop 或 git stash apply -> 恢复刚才写到一半的代码

这里涉及到了**栈的知识**。

stash 保存的内容不是只有一份，而是可以保存多份。每执行一次 `git stash`，Git 就会往 stash 栈里新增一条记录。

最新的一条通常叫：stash@{0}，数字越小，表示越新。

再旧一点的是：stash@{1}、stash@{2}

## 最常用操作

### 临时保存当前修改

最基础的命令是：

```
git stash
```

或者更推荐写成：

```
git stash push
```

执行后，Git 会把当前工作区和暂存区里的修改保存起来，然后让项目回到一个干净状态。

比如你正在开发登录功能：

```
LoginController.java   修改中
UserService.java       修改中
```

但是突然需要切换到 `hotfix` 分支修 Bug，这时候可以执行：

```
git stash push
```

然后再切换分支：

```
git checkout hotfix
```

或者：

```
git switch hotfix
```

这样当前没写完的登录功能不会被丢掉，也不会影响你去处理别的任务。

### 添加说明

如果 stash 记录多了，只看 `stash@{0}`​、`stash@{1}` 会很难分清楚每一份是干什么的。

**<u>所以更推荐加上说明：</u>**

```
git stash push -m "开发登录功能，临时保存"
```

之后查看 stash 列表时，就能看到这条说明。

这就像给临时保存的代码贴了一个便利贴，后面找起来会轻松很多。

### 查看 stash 列表

查看当前保存过的 stash：

```
git stash list
```

示例：

```
stash@{0}: On feature-login: 开发登录功能，临时保存
stash@{1}: On dev: 修改订单模块样式
stash@{2}: On main: 调整配置文件
```

这里需要注意，stash 是按照时间倒序排列的，最新的永远在最上面。

### 查看某个 stash 里改了什么

只看简略变更：

```
git stash show stash@{0}
```

如果想看具体代码差异：

```
git stash show -p stash@{0}
```

其中 `-p` 表示以 patch 的形式展示详细修改内容。

如果你 stash 记录很多，恢复之前最好先看一下，避免把不相关的旧代码恢复回来。

## 恢复

恢复 stash 主要有两个命令：`apply`​ 和 `pop`。

它们看起来都能把代码拿回来，但差别很重要。

### apply

​`git stash apply`

**恢复代码，stash 记录还保存着。**

默认恢复最新的一条：`git stash apply stash@{0}`

```
git stash apply stash@{0}
```

如果要恢复指定记录：`git stash apply stash@{1}`

也就是说，你把临时代码拿回来了，但 Git 不会自动删除那条 stash。

适合的场景是：你不确定这份 stash 恢复后是否完全正确，想先拿出来试试看。如果没问题，再手动删除。

### pop

​`git stash pop`

**恢复代码，并尝试删除这条 stash 记录。**

它相当于：`git stash apply`​ 加上 `git stash drop`

也就是说，代码恢复回来之后，这条临时记录就会从 stash 栈里移除。

不过如果恢复时发生冲突，Git 通常不会直接删除 stash 记录，这样可以避免代码丢失。

​`pop` 更适合已经确定这条 stash 就是要继续使用的场景。

### apply 和 pop 的区别

|命令|是否恢复代码|是否删除 stash 记录|适合场景|
| ------| --------------| ---------------------| ------------------------|
|​`git stash apply`|是|否|想先恢复看看，保留备份|
|​`git stash pop`|是|是|确定要恢复并继续开发|

## 删除

### 删除特定 stash

```
git stash drop stash@{0}
```

这会删除指定的一条 stash。

如果你已经用 `apply` 恢复过代码，并且确认这条 stash 不需要了，就可以手动删除。

### 清空所有 stash

```
git stash clear
```

这个命令会删除所有 stash 记录。

这个操作要谨慎，因为一旦清空，普通方式下就很难再找回来了。

一般不建议随手执行，除非你非常确定这些临时修改都不需要了。

## 注意事项

这是学习 stash 时非常容易踩坑的地方。

默认情况下：`git stash`会保存已被 Git 跟踪的文件修改，如 git add 到暂存区的修改这两类文件

但它默认不会保存新建但还没有被 Git 跟踪的文件，如.gitignore 忽略的文件，这个新文件不会被 stash 保存。

## 保存未跟踪文件

如果想把新建的、还没被 Git 跟踪的文件也一起 stash，可以使用：`git stash push -u`

-u 是 --include-untracked 的简写，表示把未跟踪文件也保存进去。

eg：`git stash push -u -m "保存登录功能和新建测试文件"`，这样新建文件也会一起被临时保存。

## 保存被忽略的文件

如果连 `.gitignore`​ 里忽略的文件也想一起保存，可以使用`git stash push -a`命令，这会把所有文件都包含进去，包括一些本地构建产物、缓存文件、临时文件也一起 stash 进去~

## stash 指定文件

有时候你并不想保存所有修改，只想 stash 某几个文件。那就在stash后面跟上具体的文件名，跟`git add`操作一致

```
git stash push -m "只保存配置文件修改" src/main/resources/application.yml
```

如果有多个文件，就用空格分开：

```
git stash push -m "保存登录相关修改" LoginController.java UserService.java
```

这样 Git 只会把指定文件的修改临时保存起来，其他文件的修改还会留在工作区。

这个操作在多人协作时很有用。比如你只想临时收起某个模块的修改，但另一个模块还要继续调试，就可以精确 stash。

## 从 stash 创建新分支

有时候你写着写着发现：这部分代码不应该继续放在当前分支了，应该单独开一个新分支。

这时可以使用：

```
git stash branch new-branch-name stash@{0}
```

执行逻辑：

1. 基于 stash 创建时所在的提交创建一个新分支
2. 切换到这个新分支
3. 把 stash 里的修改恢复出来
4. 如果成功，删除对应的 stash

这个命令适合写着写着发现分支开错了的情况。比如你本来在 `dev`​ 分支上随手改了登录功能，但后来发现应该新开一个 `feature-login`​ 分支。这时就可以先 stash，再用 `git stash branch` 把修改转移到新分支上。

## stash 冲突解决

stash 恢复时也可能产生冲突。比如你 stash 的时候修改了 `UserService.java`​，后来目标分支也修改了同一个文件的同一段代码。此时执行`git stash pop`：

```
git stash pop
```

Git 可能会提示冲突。

冲突文件里通常会出现类似结构：

```
<<<<<<< Updated upstream
当前分支中的代码
=======
stash 中恢复出来的代码
>>>>>>> Stashed changes
```

解决方式和普通 merge 冲突类似：先打开冲突文件，手动决定保留哪部分代码。修改完成后再次add冲突文件后文件执行就可以接下来的工作了。

如果你使用的是 `pop`，并且发生了冲突，要注意检查 stash 是否还在：

```
git stash list
```

如果冲突导致 stash 没有被删除，解决完之后可以手动删除：

```
git stash drop stash@{0}
```

---

## 命令总结

|命令|作用|
| ------| ---------------------------------|
|​`git stash`|临时保存当前修改|
|​`git stash push`|更推荐的 stash 写法|
|​`git stash push -m "说明"`|保存 stash 并添加说明|
|​`git stash list`|查看 stash 列表|
|​`git stash show stash@{0}`|查看某条 stash 的简略变更|
|​`git stash show -p stash@{0}`|查看某条 stash 的详细代码差异|
|​`git stash apply`|恢复最新 stash，但不删除记录|
|​`git stash apply stash@{1}`|恢复指定 stash，但不删除记录|
|​`git stash pop`|恢复最新 stash，并删除记录|
|​`git stash drop stash@{0}`|删除某条 stash|
|​`git stash clear`|清空所有 stash|
|​`git stash push -u`|保存未跟踪文件|
|​`git stash push -a`|保存所有文件，包括 ignored 文件|
|​`git stash branch new-branch stash@{0}`|从 stash 创建新分支并恢复修改|

## 理解图

![git stash](assets/git%20stash-20260515161908-u50h01u.png)

​`git stash` 本质上是 Git 提供的临时保存现场机制。它适合在代码还没完成、不想提交，但又必须切换任务时使用。它不会替代 commit，而是补充 commit 之前那段混乱、临时、半成品的开发过程。用得好的话，stash 可以让你在多个任务之间切换时更从容，不至于因为工作区不干净而手忙脚乱~

‍
