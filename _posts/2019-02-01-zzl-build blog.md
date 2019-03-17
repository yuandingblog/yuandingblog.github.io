---  
layout: post  
title: "利用github搭建一个免费的，无限流量的Blog"  
subtitle: 利用jekyll+github pages搭建个人博客  
date: 2019-02-01  
author: "张志龙"  
header-img: "img/post-bg-digital-native.jpg"  
header-mask: "0.1"  
catalog: True  
tags:  
  - git  
  - github  
  - vscode  
  - markdown  
  - jekyll  
  - 图床  
---  

>　不想再与主机服务商打交道?只用关心你的博客内容？GitHub Pages 可以运行 Jekyll, 你很简单就可以完全免费的在 GitHub 上发布网站无需数据库、评论功能，不需要不断的更新版本...  

# 前言  

jekyll是一个简单的免费的Blog生成工具，类似WordPress。但是和WordPress又有很大的不同，原因是jekyll只是一个生成静态网页的工具，不需要数据库支持。但是可以配合第三方服务,例如Disqus。最关键的是jekyll可以免费部署在Github上，而且可以绑定自己的域名。

本文将介绍如何通过jekyll基于github搭建个人blog，同时介绍如何使用微软公司推出的vscode来编辑markdown格式的blog，vscode中可以集成git、markdown、图床等插件，我们以后编写、发表博文，配置博客项目，都可以在vscode这一个工具中进行。以及如何通过本地jekyll环境来预览我们的博客系统效果。大体需要以下的步骤：  
* 注册github帐号并fork 已有jekyll模版项目  
* 在个人电脑上安装git环境并clone项目到本地  
* 安装本地jekyll环境  
* 安装vscode工具并配置必要插件  
* 修改博客项目配置并预览效果  
* 推送最终的配置和博文到github  

所需要的工具及功能如下：  
* **git**：下载、上传项目代码  
* **vscode**：修改代码，编写博客  
* **jekyll**：预览本地项目效果  

如果无法从官方下载对应的软件，可以从我的网盘下载：链接: [https://pan.baidu.com/s/1PyBArQ5RlcKPIEdchigJcw](https://pan.baidu.com/s/1PyBArQ5RlcKPIEdchigJcw) 提取码: wnwy  

# 一、注册github帐号并fork博客项目代码  

> Github Pages 是面向用户、组织和项目开放的公共静态页面搭建托管服 务，站点可以被免费托管在 Github 上，你可以选择使用 Github Pages 默 认提供的域名 github.io 或者自定义域名来发布站点。

　　首先我们需要注册一个github的帐号（注册地址：[https://github.com/](https://github.com/)），然后fork博客系统的项目代码到自己的github代码仓库中，这样我们就可以以自己的项目为基础，修改代码，上传自己的博文。如何修改代码，及如何写作博文，在后续的步骤中会有详细的介绍。  
　　按照注册要求，填写对应的用户名、密码、邮箱等信息，并通过邮箱激活注册即可，具体的注册过程略。  
　　注册好github帐号之后，访问公开的博客项目，这里我推荐使用黄玄的模板（[https://github.com/Huxpro/huxpro.github.io](https://github.com/Huxpro/huxpro.github.io)），里边集成了很多实用的功能，比如自定义域名、侧边栏、标签、评论等。我们只需对配置文件中的内容进行简单修改就可以使用了。  
　　访问到该博客项目后，点击右上角的“fork”按钮，将自动将项目代码fork到自己的github代码仓库中。  
![2019-02-01-15-23-48](http://img.zzl.yuandingsoft.com/blog/2019-02-01-15-23-48.png)  
　　
　　然后进入fork过来的项目，点击“settings”，修改项目的名称。名称的格式是　**_username_**.github.io  。注意这里的username是登录之后显示的名字，不是登录的用户名。之后等待几分钟，github将会自动创建好博客系统，并且可以通过https://**_username_**.github.io 来访问。  
　　例如我的博客的访问地址就是：[https://pekinglone.github.io/](https://pekinglone.github.io/)  
![2019-02-01-15-28-05](http://img.zzl.yuandingsoft.com/blog/2019-02-01-15-28-05.png)  

# 二、本地安装git工具并clone项目到本地  
　　项目代码是在github中，我们需要下载到本地进行修改，同时我们编写的博客文章也是需要上传到github上的，因此在本地需要安装git工具。  
　　首先需要从git的官方网站下载git：[https://git-scm.com/](https://git-scm.com/)  
* **执行安装**  
　　然后，在执行默认安装，最终可以在任意的文件夹中点击右键然后可以打开git的命令行即可。本地安装好git工具，后续可以在vscode中使用git。  
![install-git](http://img.zzl.yuandingsoft.com/blog/install-git.gif)  
　　
* **配置git全局变量**  
　　安装好git之后，在任意的文件中，点右键打开git bash命令行，在其中执行如下命令，配置本地的用户和邮箱。  
  ``` javascript  
  git config --global user.name "your name"  
  git config --global user.email "your@email.com"  
  ```  
  ![2019-02-01-16-09-07](http://img.zzl.yuandingsoft.com/blog/2019-02-01-16-09-07.png)  

* **配置访问github的ssh-key**  
　　在git bash命令行中执行ssh-keygen命令，并按三次回车，生成ssh key的公钥和私钥。  

  ``` javascript  
  ssh-keygen -t rsa -C "your@email.com"  
  ```  
  ![2019-02-01-16-14-17](http://img.zzl.yuandingsoft.com/blog/2019-02-01-16-14-17.png)  
　　生成的密钥放在 C:/user/_username_/.ssh/ 目录中，其中id_rsa是私钥，id_rsa.pub是公钥，现在需要把公钥配置到github中。  
　　用记事本打开id_rsa.pub文件，复制其中的内容，后续要粘贴到github中创建的ssh key中。  
　　登录github，点击右上角的用户图标，之后点“settings”→“SSH and GPG keys”，然后新建一个ssh key，填写一个名称，在key中填写id_rsa.pub文件的内容（该文件内容是以ssh-rsa开头），最后输入密码确定。  
![2019-02-01-16-21-43](http://img.zzl.yuandingsoft.com/blog/2019-02-01-16-21-43.png)  

* **克隆项目代码到本地**  
　　登录github，访问博客项目，点项目右上角的“clone or download”按钮，然后复制项目代码的地址，我的项目的地址是 [https://github.com/pekinglone/pekinglone.github.io.git](https://github.com/pekinglone/pekinglone.github.io.git) 。  
![2019-02-01-16-26-04](http://img.zzl.yuandingsoft.com/blog/2019-02-01-16-26-04.png)  
　　在本地新建一个用于存放项目代码的目录，进入该目录，打开git bash，输入如下命令可以将项目代码克隆到本地一个以项目名称命名的目录中。  
  ``` javascript  
  git clone https://github.com/pekinglone/pekinglone.github.io.git  
  ```  


# 三、安装本地的jekyll环境并预览项目  
　　后续在代码修改部分，我们可能修改了很多内容，但是必须要提交到github才能看到效果，我们可能修改并提交了很多次才实现了最终的效果，这个时候在github中就会有很多无意义的提交。为了避免这种尴尬的事情发生，我们可以在本地安装一个jekyll环境，每次修改完成后只需保存到本地就可以看到最终的效果。  
　　首先要下载jekyll相关的软件：[https://rubyinstaller.org/downloads/](https://rubyinstaller.org/downloads/)  
　　推荐下载两个软件，刚好可以配套使用，因为后续有很多gem包要安装，有很多因为ruby的版本问题无法安装，因此推荐使用2.3版本的ruby：  
　　        * Ruby 2.3.3 (x64)  
　　        * DevKit-mingw64-64-4.7.2-20130224-1432-sfx.exe  ：支持ruby2.0~2.3  



* **安装Ruby**  
　　安装ruby时需要注意的两个问题是，一个是安装路径中不能有空格，最好安装到根目录中，如E:\Ruby23-x64，一个是要勾选添加PATH环境变量的选项。  
![install-ruby](http://img.zzl.yuandingsoft.com/blog/install-ruby.gif)  

* **安装DevKit**  
　　1）DevKit是个自解压的文件，将其安装解压到某个文件夹，如E:\DevKit。  
　　2）进入DevKit的安装目录，打开CMD命令行，执行命令初始化config.yml文件。  
  ``` javascript  
  # 进入安装目录  
  E:  
  cd DevKit  

  # 初始化config.yml文件  
  ruby dk.rb init  

  # 使用记事本打开config.yml文件，查看文件末尾是否有 - E:\Ruby200-x64 ，正常执行完上一条命令会自动加入这一条目，如果没有的话，就手工添加。  
  notepad config.yml  

  # 安装  
  ruby dk.rb install  
  ```  

* **安装本地jekyll环境**  
　　依照如下步骤安装jekyll及所需的组件，后续可以在本地的博客项目目录中，启动jekyll服务预览博客系统。默认是通过 [http://127.0.0.1:4000](http://127.0.0.1:4000) 预览的。  
  ``` javascript  
  # 查看gem版本  
  gem -v  

  # 替换gem源，有部分gem包无法通过https下载，可以将gem源替换成http的  
  gem sources --add http://rubygems.org/ --remove https://rubygems.org/  

  # 安装jekyll  
  gem install jekyll  
  
  # 安装jekyll-paginate，否则启动jekyll会报缺少该gem包的错误，如果报其他包未安装，则安装对应的包即可  
  gem install jekyll-paginate  

  #进入本地博客项目目录  
  F:  
  cd blog\pekinglone.github.io  

  # 启动jekyll并预览博客项目效果  
  jekyll serve  
  ```  


# 四、安装VScode工具并配置插件  
　　blog是用markdown写的，每一篇博客对应一个markdown文件，vscode工具支持编辑markdown文件，因此我们可以使用vscode工具来编辑博客，同时利用vscode中的插件来提高效率，比如说git插件、图床插件等等。  

* **安装vscode**  
　　从微软官网上下载vscode工具：[https://code.visualstudio.com/](https://code.visualstudio.com/)  
　　执行安装  
![install-vscode](http://img.zzl.yuandingsoft.com/blog/install-vscode.gif)  
* **配置插件**  
　　所需安装的插件如下：  

    插件名称 | 用途  
    ---------|--------  
    git | vscode默认是支持git的，不用安装。  
    git history | 查看git提交历史的插件。  
    markdown | vscode默认是支持编辑markdown的，不用安装。  
    markdown Preview Enhanced | 如果觉得vscode自带的预览工具效果不好，可安装这个工具，在工作区的上方显示对应的功能图标。  
    markdown shortcuts | markdown语法，在工作区的上方显示对应功能的图标，或者通过右键来使用。  
    Markdown-AutoLinefeed | 可右键使用，AutoLinefeed，将在每一段最后添加空格，保证段落之间的正常换行。如果不使用该插件，可能每段完成后需要多换行几次才能编写下一段，否则换行显示不正常。  
    paste image to qiniu | 支持粘贴图片到文章中（ctrl + alt + V），并上传到七牛云，且自动生成外链连接。  
    qiniu-upload-image | 支持插入本地的图片到文章中，并上传到七牛云，且自动生成外链连接。两种方式：粘贴图片文件路径（shift + p）；选择本地图片文件（shift + o）。  
    Chinese (Simplified) Language Pack for Visual Studio Code|vscode的简体中文包，安装后界面是中文。  
  
  　　各插件的安装方式一样，点击左侧的“扩展”→搜索插件名称→“安装”。个别插件需要配置，后面会说明，其他插件可以直接使用。  
![2019-02-01-17-54-59](http://img.zzl.yuandingsoft.com/blog/2019-02-01-17-54-59.png)  

* **配置七牛云图床**  
　　本次安装了两个七牛云的插件，一个支持直接粘贴图片，一个支持上传本地的图片，两个插件配置的内容基本相同。需要注册七牛云的帐号，并在其中获取所需的配置。  
　　不同版本的vscode配置方式不同，最新版本的vscode可以通过图形界面配置，老版本的可能需要通过json文件配置，以下为json文件配置的内容，按需修改即可。  
```  
{  
    // 有效的七牛 AccessKey 签名授权，可在七牛云的“个人中心”的“密钥管理”中创建 AccessKey和SecretKey。  
    "pasteImageToQiniu.access_key": "*****************************************",  

    // 有效的七牛 SecretKey 签名授权，可在七牛云的“个人中心”的“密钥管理”中创建 AccessKey和SecretKey。  
    "pasteImageToQiniu.secret_key": "*****************************************",  

    // 七牛图片上传空间，即七牛云的“对象存储”中创建的“存储空间（bucket）”。  
    "pasteImageToQiniu.bucket": "blog",  

    // 七牛图片上传路径，参数化命名，暂时支持 ${fileName}、${mdFileName}、${date}、${dateTime}  
    // 示例：  
    //   ${fileName}-${date} -> picName-20160725.jpg  
    //   ${mdFileName}-${dateTime} -> markdownName-20170412222810.jpg  
    //   也可以加路径：如 blog/${fileName}  
    "pasteImageToQiniu.remotePath": "${fileName}",  

    // 七牛图床域名，七牛云会自动给一个30天的免费域名，过期之后不能再使用，因此需要在七牛云中“**绑定域名**”中配置一个域名的CNAME，作为长期域名使用，该域名需提前注册并备案。  
    // 如果七牛云中绑定了多个域名，需要在“**内容管理**”中将“**默认外链域名**”更改为最终对外提供服务的域名。  
    "pasteImageToQiniu.domain": "http://xxxxx.xxxx.com",  

    // 本地储存位置，粘贴的图片在本地的博客项目中会保存一份。  
    "pasteImageToQiniu.localPath":"./img"  
}  
```  
　　最新版本的vscode可在“文件”→“首选项”→“设置”中搜索 qiniu ，然后按照要求填写需要的七牛云配置信息。  
　　以下为一个配置的示例：  
![2019-02-10-19-05-47](http://img.zzl.yuandingsoft.com/blog/2019-02-10-19-05-47.png)  

# 五、修改博客项目配置并预览效果  
　　克隆到本地的博客项目需要修改配置，修改为自己的才能使用。主要修改的内容有：修改_config.yml文件的配置，修改一些网页的说明，删除原作者的博客和相关的图片等内容。  
　　在vscode左侧的“explorer”中打开项目的目录，将会列出该目录下的所有的结构和文件。  
### 5.1 修改博客项目配置_config.yml  
　　以下为_config.yml文件中需要修改的主要内容：  

``` javascript  
# Site settings  
title: Hux Blog             # 你的博客网站标题  
SEOTitle: Hux Blog          # 网页的header  
description: "Cool Blog"    # 随便说点，描述一下  
url: "https://pekinglone.github.io"  #github中博客项目的博客地址  

# SNS settings      
# 多余不需要的SNS都可以用“#”注释掉  
github_username: huxpro     # 你的github账号  
weibo_username: huxpro      # 你的微博账号，底部链接会自动更新的。  

# Build settings  
paginate: 10              # 一页你准备放几篇文章  
exclude: ["portfolio","less","node_modules","Gruntfile.js","package.json","README.md","README.zh.md"] # 在网页中忽略不展示的内容，添加“portfolio”进去，可去掉网页右上角的“**portfolio**”页面。  

# 评论区使用的disqus帐号，如果需要请改成自己的，不要用原作者的；如不需要就注释掉。  
# Disqus settings  
# disqus_username: hux  

# 网站分析，现在支持百度统计和Google Analytics。需要去官方网站注册一下，然后将返回的code贴在下面。  
# 如果不要此功能，请注释掉。  
# Baidu Analytics  
ba_track_id: 4cc1f2d8f3067386cc5cdb626a202900  

# Google Analytics  
ga_track_id: 'UA-49627206-1'            # 你用Google账号去注册一个就会给你一个这样的id  
ga_domain: huangxuan.me			# 默认的是 auto, 这里我是自定义了的域名，你如果没有自己的域名，需要改成auto。  

# 侧边的迷你个人简介栏，涉及到个人头像和个人简介。  
# Sidebar settings  
sidebar: true                           # whether or not using Sidebar.  
sidebar-about-description: "For the next quantum leap<br>离开世界之前，一切都是过程"  
sidebar-avatar: /img/avatar-hux-ny.jpg      # use absolute URL, seeing it's used in both `/` and `/about/`  

# 友情连接，按照格式修改为自己感兴趣的网站  
# Friends  
friends: [  
    {  
        title: "Foo Blog",  
        href: "http://foo.github.io/"  
    },  
    {  
        title: "Bar Blog",  
        href: "http://bar.github.io"  
    }  
]  
```  
### 5.2 定制个人blog属性  
　　由于是要搭建个人的博客系统，因此博客网页中显示的字样及背景最好进行修改，尽量避免直接使用原作者的。  
　　博客系统的主页（home）、个人简介页（about）、博文归档页面（archive），以及其他部分网页中有标题、描述、背景图片等内容是可以定制修改的.  
　　以下为几个可定制网页的对照关系：  

网页 | 对应文件  
---------|-------  
 主页（HOME） | index.html  
 个人简介页（about） | about.html，及_includes/about文件夹  
 博文归档页（archive） | archive.html  
 404页面 | 404.html  

　　以下以修改个人简介页面为例说明：  
　　打开about.html文件，主要修改如下内容，其他各个网页修改方法与之一样。  
```javascript  
# 主要修改各个网页最上端yaml配置内容  
---  
layout: page                         # 表明是网页  
title: "About"                       # 该网页的header，网页上端的主题字，以及在主页右上角上显示的字样  
description: "你是我的梦想"           # 该网页上端的说明  
header-img: "img/about-bg.jpg"       # 该网页上端的背景图  
multilingual: true  
---  
```  
　　**about网页中有评论区，如果不需要，可以删除”disqus 评论框”和 “disqus 公共JS代码”部分的代码。**  
　　另外需要修改_includes/about文件夹中的zh.md和en.md文件内容，这两个文件均是个人简介的内容，中英文两份而已。  

### 5.3 替换原作者相关的博客和图片  
　　博客系统放置博文的目录是_post，其中所有的markdown文件都是博文，正式使用前可以删除原作者的博文。  
　　放置博客系统所需的图片的目录是img，需要替换的图片主要有两种，一种是原作者的头像文件需要替换为自己的，有三张图片，大家一看就知道了；另一种是网站网页的小图标：favicon.ico，可替换为自己的。  

### 5.4 编写自己的博客  
　　在vscode中编写markdown格式的博文文件，文件名称最好使用英文，因为中文会进行转义，在url中不方便查看。  
　　文件的命名格式为：YYYY-MM-DD-name.md 。年份采用四位，月份和日期均采用双位，例如：2019-01-01-my first blog。文件名中可以有空格，在url中将自动变为“-”连接符。  
　　博文的内容为markdown格式，正常编写即可，需要主要的是博文的头部必须添加yaml格式的配置，用于定义博文的标题、小标题、作者、标签等内容，具体如下：  
```javascript  
---  
layout: post                          # 表明是博文  
title: "如何搭建个人博客系统"           # 博文的标题  
subtitle: 利用github搭建私有博客系统    # 博文的小标题  
date: 2019-02-01                      # 博文的发表日期，此日期决定主页上博文的先后顺序  
author: "张志龙"                       # 博文的作者  
header-img: "img/post-bg-cka.jpg"     # 博文页面上端的背景图片  
header-mask: "0.1"                    # 博文页面上端的背景图片的亮度，数值越大越黑暗  
catalog: True                         # 开启catalog，将在博文侧边展示博文的结构  
tags:                                 # 本篇博文的标签，该标签将在archive页面中对博文进行分类  
  - git  
  - github  
  - vscode  
  - markdown  
  - jekyll  
  - 图床  
---  

```  


### 5.5 预览确认效果，并将项目代码同步到github  
　　打开[http://127.0.0.1:4000](http://127.0.0.1:4000)，预览博客系统网页修改的效果，当所有的修改均完成后，便可以将项目代码同步到github中。  
　　可以使用git命令的方式将代码同步到github，也可以使用vscode同步。  
　　在vscode左侧的“source control”中可以看到当前项目中文件的状态，点击“**+**”将修改提交到暂存区（相当于 git add .），点击“**√**”将暂存区的内容提交到本地代码仓库中（相当于 git commit），最后点“推送”可将本地的代码仓库推送到远端的github代码仓库中，之后自己的博文就会发布到github上了。  
　　大家可以通过访问自己的地址查看blog，我的blog [https://pekinglone.github.io/](https://pekinglone.github.io/) 。  
