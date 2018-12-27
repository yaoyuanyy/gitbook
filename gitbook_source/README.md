# Introduction 
这里是什么
```
这是一个平日开发过程中记录技术点的小营地，涵盖了java、springboot、springcloud、linux、mysql等，同时记录了解决线上问题的轨迹和平日工作中的技巧
```

也想弄个书籍
```
GitBook 是一个基于 Node.js 的命令行工具，可使用 Github/Git 和 Markdown 来制作精美的电子书。
1、安装node
$ brew install node

2、安装gitbook
$ npm install gitbook-cli -g

3、初始化项目
$ gitbook init
会自动创建一个使用gitbook init后会自动生成两个文件README.md和SUMMARY.md
README.md使用过git的都知道这个文件
SUMMARY.md就是自己要写文章章节目录 

如果你想本地查看效果，运行gitbook serve命令后访问：// 直接访问localhost:4000
4、启动服务(可选)
$ gitbook serve


如果你想构建后推到gitbub上
首先gitbub创建一个库备用，然后本地创建目录如下(本人小技巧，一个放源码另一个放生成的文件，防止gitbook build 后源码被覆盖)
$ cd ~/skyler/project/gitbook
$ mkdir gitbook_source
$ mkdir gitbook_web
$ tree
gitbook
├── gitbook_source
└── gitbook_web
运行gitbook build 构建服务，具体如下
4、构建服务(可选)
 格式：gitbook build 源目录 输出目录
$ cd ~/skyler/project/gitbook 
$ gitbook build gitbook_source gitbook_web
生成后的文件目录树为底部格式
$ git push origin github_addr 
注意：github库需要有个分支：gh-pages
$ curl https://yaoyuanyy.github.io/gitbook/gitbook_web/
```


gitbook目录结构树
```
skyler@skylerdeMacBook-Pro ~/skyler/project $ tree gitbook
gitbook
├── gitbook_source
│   ├── 01.md
│   ├── README.md
│   └── SUMMARY.md
└── gitbook_web
    ├── 01.html
    ├── gitbook
    │   ├── fonts
    │   │   └── fontawesome
    │   │       ├── FontAwesome.otf
    │   │       ├── fontawesome-webfont.eot
    │   │       ├── fontawesome-webfont.svg
    │   │       ├── fontawesome-webfont.ttf
    │   │       ├── fontawesome-webfont.woff
    │   │       └── fontawesome-webfont.woff2
    │   ├── gitbook-plugin-fontsettings
    │   │   ├── fontsettings.js
    │   │   └── website.css
    │   ├── gitbook-plugin-highlight
    │   │   ├── ebook.css
    │   │   └── website.css
    │   ├── gitbook-plugin-lunr
    │   │   ├── lunr.min.js
    │   │   └── search-lunr.js
    │   ├── gitbook-plugin-search
    │   │   ├── lunr.min.js
    │   │   ├── search-engine.js
    │   │   ├── search.css
    │   │   └── search.js
    │   ├── gitbook-plugin-sharing
    │   │   └── buttons.js
    │   ├── gitbook.js
    │   ├── images
    │   │   ├── apple-touch-icon-precomposed-152.png
    │   │   └── favicon.ico
    │   ├── style.css
    │   └── theme.js
    ├── index.html
    └── search_index.json
```
 