# Github 、Gitee 自动同步脚本
> 相关笔记：[[Git|Git 知识总结]]


# 解决 GitHub Actions 同步 Gitee 过程中遇到的一些问题

## 前言

由于是在CSDN上更新博客，随着不断学习，想了想应该不应该局限于这一个平台

于是萌生出搭建一个属于自己博客网站，最终决定采用Hugo + Github

Hugo工具负责生成静态页面，然后在Github上来部署静态页面，免费搭建出自己的博客

b站上有非常优质的up来手把手搭建博客网站，我这里看的是莱特雷

<iframe sandbox="allow-top-navigation-by-user-activation allow-same-origin allow-forms allow-scripts allow-popups" src="https://player.bilibili.com/player.html?bvid=BV1bovfeaEtQ&amp;page=1&amp;high_quality=1&amp;as_wide=1&amp;allowfullscreen=true&amp;autoplay=0&amp;spm_id_from=333.1007.top_right_bar_window_custom_collection.content.click&amp;vd_source=e01647f259a0769d95e99766835e2fde" border="0" frameborder="no" framespacing="0" allowfullscreen="true" style=""></iframe>

## 思路

up实现了GitHub自动部署的功能，但由于结合实际情况，我主要的代码存在Gitee的仓库中，在Github上更新不能在我的码云主页上有完整体现

而用普通的镜像仓库不能做到同步的功能，Gitee导入Github仓库功能本质上是一个一次性克隆的仓库。如果想在Gitee上看到还需要手动同步，非常低效且麻烦

😶‍🌫️所以我想新增一个懒功能：在GitHub更新Hugo博客后，自动同步到Gitee，并点亮Gitee的 “小绿点”，同时也是为了解决国内访问速度问题

所以就继续完善莱特雷博主写的GitHub Action，每次 Push 到 GitHub 后，自动推送到 Gitee

## 技术实现

配置了 **GitHub Actions**，利用 **SSH 协议** 将代码自动同步分发到 Gitee 仓库，实现了 ”一次提交，双端自动部署“ 的高效工作流，要注意：自动化脚本都不能生成密码，否则就不能自动执行了

1. SSH密钥生成
2. 密钥与 Token 配置，生成后，需要分发到不同平台：

   - **GitHub (Settings ->**  **Secrets ->**  **Actions):**

     - ​`GITEE_RSA_PRIVATE_KEY`​: 填入 **私钥** (生成的 `gitee_id_rsa`​ 文件内容，是以 `-----BEGIN RSA PRIVATE KEY-----` 开头的格式)
     - ​`GITEE_TOKEN`​: 填入 **Gitee 私人令牌** (在 Gitee 设置 -\> 私人令牌中生成，需勾选 `projects`​ 和 `hook` 权限)
   - **Gitee (设置 ->**  **SSH 公钥):**

     - 填入**公钥** (生成的 `gitee_id_rsa.pub` 文件内容)
3. 填写配置文件

## 遇到的坑

### 坑一：SSH的密钥格式问题

在同步Gitee仓库的时候出现了报错，虽然GitHub上可以完美推送到仓库中

![报错日志](assets/image-20251208201034-ig9ur2f.png "报错日志")

**原因：**

1. 由于我是直接使用默认的`ssh-keygen`​生成密钥，生成的是新版 OpenSSH 格式（以 `BEGIN OPENSSH PRIVATE KEY`​ 开头），而这个脚本环境比较老，只支持`PEM`格式的私钥

2. **配置在 GitHub Secrets 里的私钥格式，GitHub Actions 的环境无法识别，是格式不兼容的问题**

**解决方法：**

1. 重新生成 PEM 格式密钥，在本地终端执行

   ```bash
   # 在你的本地终端执行（不要在服务器上）
   # 注意：文件名可以改，不要覆盖了你原本的id_rsa
   ssh-keygen -t rsa -b 4096 -C "你的邮箱地址.com" -f ./gitee_id_rsa -m PEM
   ```
2. 更新Github配置（更新私钥）

   - 回到你 GitHub 仓库的 **Settings ->**  **Secrets and variables ->**  **Actions**
   - 找到 `GITEE_RSA_PRIVATE_KEY`，更新新的私钥
   - 更新后记得保存
3. 更新Gitee配置（更新公钥）

   - 打开生成的 `gitee_id_rsa.pub` (公钥)
   - 去 Gitee -\> 设置 -\> SSH 公钥
   - 删除之前添加的那个报错的公钥，格式不兼容留着也没有用了
   - 添加这个新的公钥

### 坑二：仓库名称映射问题

解决了密钥问题后，同步依然失败，我怀疑是同步到了错误的仓库

**原因：**

由于前期的维护问题，我的Github上的用户名与Gitee上的用户名并不一致，仓库名也有细微区别，这就使得<u>**同步工作变得困难**</u>

**解决方法：**

经排查，我发现GitHub上博客仓库名是 `Jul1en-Lin.github.io`​， Gitee 那边在由于创建仓库的时，仓库访问URL路径默认是仓库名的小写形式，是`jul1en-lin.github.io`，两边名字不一样，导致不能正确同步

在配置文件中`static_list`​列表正确填写上Gitee是正确的仓库访问路径：`jul1en-lin.github.io`，问题就解决了~

![image](assets/image-20251208203516-fotvncr.png)

修改后的配置文件如下，注意要严格对应仓库路径 **==（笔记中是原版的配置文件，不需要更改，用于备份保存）==**

```yaml
name: deploy

# 代码提交到main分支时触发github action
on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
        - name: Checkout
          uses: actions/checkout@v4
          with:
              fetch-depth: 0

        - name: Setup Hugo
          uses: peaceiris/actions-hugo@v3
          with:
              hugo-version: "latest"
              extended: true

        - name: Build Web
          run: hugo -D

          # 推送到Github仓库的Jul1en-Lin.github.io 仓库
        - name: Deploy Web
          uses: peaceiris/actions-gh-pages@v4
          with:
              PERSONAL_TOKEN: ${{ secrets.HUGO_TOKEN }}
              EXTERNAL_REPOSITORY: Jul1en-Lin/Jul1en-Lin.github.io
              PUBLISH_BRANCH: main
              PUBLISH_DIR: ./public
              commit_message: auto deploy
              # 关键：为了 Gitee 小绿点，这里必须填你 Gitee 绑定的邮箱和昵称
              # 这样生成的 Commit 会归属到你名下，同步过去才有绿点。
              user_name: 'Jul1en-Lin' 
              user_email: '1577045857@qq.com'

        # 同步到 Gitee
        - name: Sync to Gitee
          uses: Yikun/hub-mirror-action@master
          if: always() # 保证即使上面步骤偶尔抽风，这一步也尝试执行
          with:
            #  GitHub 用户名
            src: github/Jul1en-Lin
            #  Gitee 用户名
            dst: gitee/Cary_C
            # 只需要同步存放静态页面的那个仓库
            dst_key: ${{ secrets.GITEE_RSA_PRIVATE_KEY }}
            dst_token: ${{ secrets.GITEE_TOKEN }} # 需要去 Gitee 生成一个私人令牌(Access Token)
            static_list: "jul1en-lin.github.io" # 这里填你要同步的仓库名
            force_update: true
            debug: true
```

## 总结

- 了解到 SSH 密钥还会有不同格式的区别，熟悉了 GitHub Actions 的配置，这也是第一次接触GitHub Actions，配置文件也是第一次接触，但是静下心来看还是能看的懂并尝试解决的🤗
- 遇到报错不要慌，仔细看 Log。**培养能仔细查看报错日志是非常重要的，遇到的问题可能千奇百怪，如果能看懂报错日志就可以定位出现问题的原因了，这样以后遇到问题也不会害怕。毕竟**​**==授人以鱼不如授人以渔~==**
- 第一次接触自动化部署，虽然配置时麻烦，但配好并且解决问题后的感觉真的很爽，满满自豪感！！！！🥳😽

‍
