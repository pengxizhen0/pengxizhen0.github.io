### 如何新建文章

在根目录下执行`hexo new post (文章标题)`



### 如何管理源码和发布

hexo发布出去的话，是静态网站，只包含内容。我们需要把包括npm、hexo主题在内的相关配置上传。

源码在`develop`分支，静态内容发布在`master`分支。

在`develop`分支写完文章后，可以`git push`源码到Github。

然后使用`hexo clean`、`hexo generate`、`hexo deploy`发布静态内容至`master`分支。

然后访问https://season-peng.github.io/ 即可看到新网页。

