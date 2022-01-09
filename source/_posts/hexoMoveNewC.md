---
title: Move hexo to new computer
---

## Quick Start

### Create a new branch

旧电脑上传分支
首先，先在github上新建一个source分支
然后在这个仓库的settings中，选择默认分支为hexo分支（这样每次同步的时候就不用指定分支，比较方便）。
然后在本地的任意目录下，打开git bash，

git clone git@github.com:***/***.github.io.git source
将其克隆到本地，因为默认分支已经设成了source，所以clone时只clone了source。
在source目录中，把除了.git 文件夹外的所有文件都删掉
讨论下哪些文件是必须拷贝的：
首先是之前自己修改的文件，像站点配置_config.yml，theme文件夹里面的主题，以及source里面自己写的博客文件，这些肯定要拷贝的。
除此之外，还有三个文件需要有，就是scaffolds文件夹（文章的模板）、package.json（说明使用哪些包）和.gitignore（限定在提交的时候哪些文件可以忽略）。
其实，这三个文件不是我们修改的，所以即使丢失了，也没有关系，我们可以建立一个新的文件夹，然后在里面执行hexo init，就会生成这三个文件，我们只需要将它们拷贝过来使用即可。
总结：_config.yml，theme/，source/，scaffolds/，package.json，.gitignore，是需要拷贝的。
再讨论下哪些文件是不必拷贝的，或者说可以删除的：
首先是.git文件，无论是在站点根目录下，还是主题目录下的.git文件，都可以删掉。
然后是文件夹node_modules（在用npm install会重新生成），public（这个在用hexo g时会重新生成），.deploy_git文件夹（在使用hexo d时也会重新生成），db.json文件。
其实上面这些文件也就是.gitignore文件里面记载的可以忽略的内容。总结：.git/，node_modules/，public/，.deploy_git/，db.json文件需要删除。


### push to new branch

而后
git add .
git commit –m "add branch"
git push 
这样就上传完了，可以去你的github上看一看hexo分支有没有上传上去，其中node_modules、public、db.json已经被忽略掉了，没有关系，不需要上传的，因为在别的电脑上需要重新输入命令安装 。


### 更换电脑操作

安装git
设置git全局邮箱和用户名
设置ssh key
安装nodejs
安装hexo
npm install hexo-cli -g
直接在任意文件夹下，
git clone git@……………… source
cd source
npm install
npm install hexo-deployer-git --save

需要使用git push把源文件推到分支上
$ git add .
$ git commit -m "xxxx"
$ git push origin hexo

### Deploy to remote sites
生成，部署：
hexo g
hexo s 
本地查看
hexo d




