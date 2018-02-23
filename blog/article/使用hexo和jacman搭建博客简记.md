---
title: 使用hexo和jacman搭建博客简记
date: 2016/06/23 21:12:22
toc: false
list_number: false
categories:
- 教程
tags:
- hexo jacman
---

## 写一篇博客
1.  在创建博客目录的`.blog/source/_posts`文件夹中创建一个md文件.
2. 支持markdown写博客, 不同之处在于文件开头需要添加一个描述:
   ```
        ---
        title: 记一次22亿大数据分析处理踩坑经历
        date: 2017/3/23 21:12:22
        toc: false
        list_number: false
        categories:
        - 脚本
        tags:
        - hive
        - mysql
        - shell
        ---
    ```
其中, `toc: false` 表示不为该篇文章创建目录.
3. md中添加图片
首先将图片放在source目录的image目录下，然后md中引用：`![](/images/test.jpg)`
4. 站内链接
格式：`[Google](http://www.google.com/)`
5. 资源
[jacman](http://jacman.wuchong.me)，[hexo](https://hexo.io/zh-cn/)

- - - - -

## 常用命令
1. `hexo s` 本地开启http服务，查看效果。
2. `hexo generate -d` 生成静态文件，并推送到网站上。`hexo -g depploy `部署之前先生成静态文件。
3. `hexo g` 生成静态文件。
4. `hexo d` 部署到网站上。
5. `hexo clean` 清除缓存文件和已生成的静态文件。

## hexo安装使用
主要记录安装问题及解决:
1. 二进制安装nodjs, 安装hexo后,全局下不能识别hexo命令, 只有在nodejs的`/lib/node_modules/hexo-cli/bin`目录下执行hexo命令才生效.
解决: 需要添加软连接: `sudo ln -s /usr/develop/node/node-v4.5.0/lib/node_modules/hexo-cli/bin/hexo /usr/local/bin/hexo`

2. 每次推送github都要数据用户名密码.
解决: 是因为在`_config.yml`文件中配置git地址是`http`推送, 将其改为`ssh`推送,并删掉`.deploy_git`目录和`hexo clean`,
并且保证博客根目录有访问权限(每次加sudo麻烦,且`sudo hexo d`还可以能为报错).最后,`hexo g`, `hexo d`看是否还需输入用户名密码.

3. 页面中英文夹杂, 比如目录`contents` ，分类`categories`，友情链接 `links`
解决：全局`_config.yml`中设定language为`zh-CN`

4. 检查软件安装
总共需要安装的软件： nodjs，npm，hexo，`npm install hexo-deployer-git --save`上传github插件，nrm选装（npm源切换）
最后，根据教程：`https://hexo.io/zh-cn/` 和 `https://github.com/wuchong/jacman/blob/master/README_zh.md` 来一步一步安装。


5. SEO优化：http://www.dajipai.cc/

## 配置域名
阿里云，腾讯云，百度云都可以购买域名，购买域名为：huzongzhe.cn
1. 认证，信息设置完成后，进入管理界面，设置域名解析，让域名跳转到指定Ip去。
2. 添加两个解析：`@`和`www`类型都是A类，Ip为`151.101.88.133`这个就是github的ip。
`@`和`www`表示支持`huzongzhe.cn`和`www.huzongzhe.cn`额域名解析
3. 在博客目录的source目录下添加一个CNAME文件（我的是`./blog/source`），里面填入huzongzhe.cn。
完成！