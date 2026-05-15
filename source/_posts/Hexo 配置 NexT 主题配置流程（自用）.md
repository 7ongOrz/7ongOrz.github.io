---
title: Hexo 配置 NexT 主题配置流程（自用）
author: Ahtong
top: false
cover: false
toc: true
abbrlink: d2354a7c
date: 2024-02-01 23:42:13
tags:
  - Hexo
  - NexT
categories: 杂项
description:
img:
coverImg:
password:
---

## Hexo 部署
### docker 版本一键安装
采用 [appotry](https://github.com/appotry) 的docker版本

```bash
docker create --name=hexo \
  -e HEXO_SERVER_PORT=4000 \
  -e GIT_USER="" \
  -e GIT_EMAIL="" \
  -v /mnt/hexo:/app \
  -p 4000:4000 \
  bloodstar/hexo
```

### SSH 部署

> **Docker会自动随机生成ssh key** 在 /app/.ssh 目录下面。
> 1.将**SSH** 公钥复制到剪贴板。
> 2.在任何页面的右上角，单击您的个人资料照片，然后单击Settings（设置）。
> 3.在用户设置侧边栏中，单击**SSH** and GPG keys（**SSH** 和GPG 密钥）。
> 4.单击New **SSH** key（新**SSH** 密钥）或Add **SSH** key（添加**SSH** 密钥）。

### 运行

```bash
docker exec -it hexo /bin/bash
```

### 常用命令

```bash
hexo server #启动本地服务器，用于预览主题。Hexo 会监视文件变动并自动更新，除修改站点配置文件外,无须重启服务器,直接刷新网页即可生效。
hexo server -s #以静态模式启动
hexo server -p 4000 #更改访问端口 (默认端口为 5000，’ctrl + c’关闭 server)
hexo server -i IP地址 #自定义 IP
hexo clean #清除缓存 ,网页正常情况下可以忽略此条命令,执行该指令后,会删掉站点根目录下的 public 文件夹
hexo g #生成静态网页 (执行 $ hexo g后会在站点根目录下生成 public 文件夹, hexo 会将”/blog/source/“ 下面的.md 后缀的文件编译为.html 后缀的文件,存放在”/blog/public/ “ 路径下)
hexo d #自动生成网站静态文件，并将本地数据部署到设定的仓库(如 github)
hexo init 文件夹名称 #初始化 XX 文件夹名称
npm update hexo -g#升级
npm install hexo -g #安装
node -v #查看 node.js 版本号
npm -v #查看 npm 版本号
git --version #查看 git 版本号
hexo -v #查看 hexo 版本号
hexo new page “music” #新增页面music
hexo new post “文章名称” #新增文章
```

<!--more-->

## Hexo 配置

### 下载主题

```bash
git clone https://github.com/next-theme/hexo-theme-next themes/next
```

### 切换主题

修改 Hexo 根目录下的`_config.yml`的`theme`的值：`theme: next`

### 永久链接修改成短链

用 [hexo-abbrlink](https://github.com/rozbo/hexo-abbrlink) 生成静态文章链接。

安装：

```bash
npm install hexo-abbrlink --save
```

修改 Hexo 根目录下的`_config.yml`文件中的如下内容:

```bash
url: https://blog.tongorz.top
# permalink: :year/:month/:day/:title/
# article之前不加:，不然会生成一个叫 undefined 文件夹
permalink: article/:year:month:day:abbrlink.html
abbrlink:
  alg: crc32  # 算法：crc16(default) and crc32
  rep: hex    # 进制：dec(default) and hex
permalink_defaults:
```

### 文章设置

```bash
# Writing
new_post_name: :title.md         # 新文章的文件名称
default_layout: post             # 预设布局
auto_spacing: false              # 在中文和英文之间加入空格
titlecase: false                 # 把标题转换为 title case
external_link:                   # 在新标签中打开链接
  enable: true                   # 在新标签中打开链接
  field: site                    # 对整个网站 (site) 生效或仅对文章 (post) 生效
  exclude: ''                    # 需要排除的域名。主域名和子域名如 www 需分别配置 []
filename_case: 0                 # 把文件名称转换为 (1) 小写或 (2) 大写
render_drafts: false             # 显示草稿，默认为：false
post_asset_folder: true          # 启动 Asset 文件夹
relative_link: false             # 把链接改为与根目录的相对位址
future: true                     # 显示未来的文章
```

### 一键部署到 github

```bash
npm install hexo-deployer-git --save
```

修改 Hexo 根目录下的`_config.yml`文件中的如下内容：

```bash
deploy:
  type: git
  repo: git@github.com:
  branch: master
  ignore_hidden: false
```

### 配置代码样式

修改 Hexo 根目录下的`_config.yml`文件中的如下内容：

```bash
code:
  lang: true     # 代码块是否显示名称
  copy: true     # 代码块是否可复制
  shrink: false  # 代码块是否可以收缩
  break: false   # 代码是否折行
```

## NexT 主题配置

### 布局设置

切换主题，在`NexT _config.yml`文件中修改：

```bash
# Schemes
# scheme: Muse
# scheme: Mist
# scheme: Pisces
scheme: Gemini
```

### 菜单栏设置

加入新的菜单栏，已有的取消注释，添加没有的需要修改两部分文件，`_config.yml`和`languages`。

1. 在`themes/next/_config.yml`文件中修改：

   ```bash
   # Usage: `Key: /link/ || icon`
   # Key is the name of menu item. If the translation for this item is available, the translated text will be loaded, otherwise the Key name will be used. Key is case-sensitive.
   # Value before `||` delimiter is the target link, value after `||` delimiter is the name of Font Awesome icon.
   # External url should start with http:// or https://
   menu:
     home: / || fa fa-home
     #about: /about/ || fa fa-user
     tags: /tags/ || fa fa-tags
     categories: /categories/ || fa fa-th
     archives: /archives/ || fa fa-archive
     resources: /resources/ || fa fa-download # 加入新菜单
     #schedule: /schedule/ || fa fa-calendar
     #sitemap: /sitemap.xml || fa fa-sitemap
     #commonweal: /404/ || fa fa-heartbeat
   ```

2. 在`/themes/next/languages`路径下的`zh-CN.yml`和`en.yml`文件中分别修改：

   ```bash
   menu:
     home: 首页
     archives: 归档
     categories: 分类
     tags: 标签
     about: 关于
     resources: 资源 # 加入翻译
     search: 搜索
     schedule: 日程表
     sitemap: 站点地图
     commonweal: 公益 404
   ```

   ```bash
   menu:
     home: Home
     archives: Archives
     categories: Categories
     tags: Tags
     about: About
     resources: Resources # 加入翻译
     search: Search
     schedule: Schedule
     sitemap: Sitemap
     commonweal: Commonweal 404
   ```

在根目录下输入如下代码：

```bash
hexo new page "categories"
hexo new page "tags"
hexo new page "resources"
```

此时在根目录的 sources 文件夹下会生成 categories、tags、resources 三个文件，每个文件中有一个 `index.md` 文件，修改内容分别如下：

```bash
---
title: categories
date: 2024-01-31 12:12:26
type: "categories"
layout: "page"
comments: false
---
```

```bash
---
title: tags
date: 2024-01-31 12:12:38
type: "tags"
layout: "page"
comments: false
---
```

```bash
---
title: resources
date: 2024-01-31 12:12:50
type: "resources"
layout: "page"
comments: false
---
```

如果有启用评论，默认页面带有评论。需要关闭的话，添加字段 comments 并将值设置为 false。

### 设置建站时间

在`themes/next/_config.yml`文件中修改：

```bash
footer:
  # Specify the year when the site was setup. If not defined, current year will be used.
  since: 2024
```

### 新建文章模板修改

修改 Hexo 根目录下的`scaffolds/post.md`文件中的如下内容：

{% raw %}
```bash
---
title: {{ title }}
date: {{ date }}
author:
tags:
categories:
description:
img:
coverImg:
top: false
cover: false
toc: true
#mathjax: false
password:
---
```
{% endraw %}

### 代码高亮设置

需要修改两部分文件， Hexo 根目录下 `_config.yml` 文件还有主题下的`themes/next/_config.yml`文件。

1. 修改 Hexo 根目录下的`_config.yml`文件中的如下内容：

   ```bash
   syntax_highlighter: highlight.js
   highlight:                       # 代码块的设置
     enable: true                   # 开启代码块高亮
     line_number: true              # 显示行数
     auto_detect: false             # 如果未指定语言，则启用自动检测
     tab_replace: ''                # 用 n 个空格替换 tabs；如果值为空，则不会替换 tabs
   prismjs:
     enable: false
     preprocess: true
     line_number: true
     tab_replace: ''
   ```

2. 在`themes/next/_config.yml`文件中修改：

   ```bash
   codeblock:
     # Code Highlight theme
     # All available themes: https://theme-next.js.org/highlight/
     theme:
       light: stackoverflow-light
       dark: stackoverflow-dark
     prism:
       light: prism
       dark: prism-dark
     # Add copy button on codeblock
     # 打开一键复制按钮
     copy_button:
       enable: true
       # Available values: default | flat | mac
       style: mac
     # Fold code block
     fold:
       enable: false
       height: 500
   ```

###  版权信息

在`themes/next/_config.yml`文件中修改：

```bash
# Creative Commons 4.0 International License.
# See: https://creativecommons.org/about/cclicenses/
creative_commons:
  # Available values: by | by-nc | by-nc-nd | by-nc-sa | by-nd | by-sa | cc-zero
  license: by-nc-sa
  # Available values: big | small
  size: small
  sidebar: false # 表示是否显示在侧边栏
  post: true # 是否在文章底部显示
  # You can set a language value if you prefer a translated version of CC license, e.g. deed.zh
  # CC licenses are available in 39 languages, you can find the specific and correct abbreviation you need on https://creativecommons.org
  language:
```

### 加载进度条

在`themes/next/_config.yml`文件中修改：

```bash
# Reading progress bar
reading_progress:
  enable: true
  # Available values: left | right
  start_at: left
  # Available values: top | bottom
  position: top
  reversed: false
  color: "#37c6c0"
  height: 3px
```

### 配置MathJax

1. 卸载`hexo-math`：` npm un hexo-math`

2. 下载pandoc：

   ```bash
   brew install Pandoc
   ```

2. 替换渲染：
   ```bash
   npm uninstall hexo-renderer-marked --save
   npm install hexo-renderer-pandoc --save
   ```

4. 在`themes/next/_config.yml`文件中修改：

   ```bash
   math:
     # Default (false) will load mathjax / katex script on demand.
     # That is it only render those page which has `mathjax: true` in front-matter.
     # If you set it to true, it will load mathjax / katex script EVERY PAGE.
     every_page: true

     mathjax:
       enable: true
       # Available values: none | ams | all
       tags: none

     katex:
       enable: false
       # See: https://github.com/KaTeX/KaTeX/tree/master/contrib/copy-tex
       copy_tex: false
   ```

   `every_page`表示是否自动渲染每一页，如果为`false`就只渲染配置块中包含`mathjax: true`的文章。

   在 Hexo 根目录下的`_config.yml`文件中添加如下内容：

   ```bash
   mathjax:
     tags: none # 或 'ams' 或 'all'
     single_dollars: true # 启用单个美元符号作为内联（行内）数学公式定界符
     cjk_width: 0.9 # 相对 CJK 字符宽度
     normal_width: 0.6 # 相对正常（等宽）宽度
     append_css: true # 将 CSS 添加到每个页面
     every_page: true # 如果为 true，那么无论每篇文章的前题中的 `mathjax` 设置如何，每页都将由 mathjax 呈现
   ```

### 搜索服务

下载：

```bash
npm install hexo-generator-searchdb --save
```

在 Hexo 根目录下的`_config.yml`文件中添加如下内容：

```bash
search:
  path: search.xml
  field: post
  content: true
  format: html
```

在`themes/next/_config.yml`文件中修改：

```bash
# Local Search
# Dependencies: https://github.com/next-theme/hexo-generator-searchdb
local_search:
  enable: true
  # If auto, trigger search by changing input.
  # If manual, trigger search by pressing enter key or search button.
  trigger: auto
  # Show top n results per article, show all results by setting to -1
  top_n_per_article: 1
  # Unescape html strings to the readable one.
  unescape: false
  # Preload the search data when the page loads.
  preload: false
```

### 字数统计

使用 [hexo-word-counter](https://github.com/next-theme/hexo-word-counter) 统计文章的字数以及预期阅读时间，完成配置后，可以在每篇文章开头和页面底部显示字数和阅读时间。

下载：

```bash
npm install hexo-word-counter
```

在 Hexo 根目录下的`_config.yml`文件中添加如下内容：

```bash
#显示文章字数和阅读时长
symbols_count_time:
  symbols: true                # 文章字数统计
  time: true                   # 文章阅读时长
  total_symbols: true          # 站点总字数统计
  total_time: true             # 站点总阅读时长
  exclude_codeblock: false     # 排除代码字数统计
  awl: 2
  wpm: 300
  suffix: "mins."
```

在`themes/next/_config.yml`文件中修改：

```bash
# Post wordcount display settings
# Dependencies: https://github.com/next-theme/hexo-word-counter
symbols_count_time:
  separated_meta: true
  item_text_total: true
```

> 如果文章中大多中文，那么设置`awl`为`2`，`wpm`为`300`比较合适

### 修改文章底部的带 # 号的标签

```bash
# Use icon instead of the symbol # to indicate the tag at the bottom of the post
tag_icon: true
```

### 配置返回顶部

在`themes/next/_config.yml`文件中修改：

```bash
back2top:
  enable: true
  # Back to top in sidebar.
  sidebar: false
  # Scroll percent label in b2t button.
  scrollpercent: true
```

### 不蒜子统计

在`themes/next/_config.yml`文件中修改：

```bash
# Show Views / Visitors of the website / page with busuanzi.
# For more information: http://ibruce.info/2015/04/04/busuanzi/
busuanzi_count:
  enable: true
  total_visitors: true
  total_visitors_icon: fa fa-user
  total_views: true
  total_views_icon: fa fa-eye
  post_views: true
  post_views_icon: far fa-eye
```

在`next/layout/_third-party/statistics/busuanzi-counter.njk`文件中修改为第三方 API：

{% raw %}
```bash
{%- if theme.busuanzi_count.enable %}
  <script{{ pjax }} async src="https://busuanzi.icodeq.com/busuanzi.pure.mini.js"></script>
{%- endif %}
```
{% endraw %}

### 添加 Canvas_nest 动态背景

根据[官方文档](https://github.com/theme-next/theme-next-canvas-nest)，在`hexo/source/_data`的目录下新建一个`footer.njk`文件（如果`_data`目录不存在则创建一个）。

在`themes/next/_config.yml`文件中修改：

```bash
# Define custom file paths.
# Create your custom files in site directory `source/_data` and uncomment needed files below.
custom_file_path:
  #head: source/_data/head.njk
  #header: source/_data/header.njk
  #sidebar: source/_data/sidebar.njk
  #postMeta: source/_data/post-meta.njk
  #postBodyStart: source/_data/post-body-start.njk
  #postBodyEnd: source/_data/post-body-end.njk
  footer: source/_data/footer.njk
  #bodyEnd: source/_data/body-end.njk
  #variable: source/_data/variables.styl
  #mixin: source/_data/mixins.styl
  #style: source/_data/styles.styl
```

最后在配置文件里新增一段`canvas_nest: true`。

### 添加 Gitalk 评论系统

[Gitalk](https://github.com/gitalk/gitalk/blob/master/readme-cn.md)是一个基于`github`开发的评论插件，它将文章评论以`issues`形式保存在`github`仓库中。

注册github应用：

进入`github`注册页面：[Register a new OAuth application](https://github.com/settings/applications/new)

- `Application name`：应用名
- `Homepage URL`：网站地址
- `Application description`：应用描述
- `Authorization callback URL`：网站地址

注册成功后会生成`Client ID`和`Client Secret`

在`themes/next/_config.yml`文件中修改：

```bash
# Gitalk
# For more information: https://gitalk.github.io
gitalk:
  enable: true
  github_id:  # GitHub repo owner
  repo:  # Repository name to store issues
  client_id:  # GitHub Application Client ID
  client_secret:  # GitHub Application Client Secret
  # admin_user要填入有仓库权限的账号
  admin_user:  # GitHub repo owner and collaborators, only these guys can initialize gitHub issues
  distraction_free_mode: true # Facebook-like distraction free mode
  # When the official proxy is not available, you can change it to your own proxy address
  # proxy: https://cors-anywhere.azm.workers.dev/https://github.com/login/oauth/access_token # This is official proxy address
  # Gitalk's display language depends on user's browser or system environment
  # If you want everyone visiting your site to see a uniform language, you can set a force language value
  # Available values: en | es-ES | fr | ru | zh-CN | zh-TW
  language: zh-CN
```

### 添加网站运行时间

方法来源于[yifanstar](https://yifanstar.top/2020/07/19/hexo-blog-creat/)

在`themes/next/layout/_partials`目录下新建一个`time.njk`文件，添加如下代码：

{% raw %}
```javascript
{% if theme.footer.site_runtime.enable %}
  <script src="https://cdn.jsdelivr.net/npm/moment@2.22.2/moment.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/moment-precise-range-plugin@1.3.0/moment-precise-range.min.js"></script>
  <script>
    function timer() {
      var ages = moment.preciseDiff(moment(),moment({{ theme.footer.site_runtime.since }},"YYYYMMDDhmmss"));
      ages = ages.replace(/years?/, "年");
      ages = ages.replace(/months?/, "月");
      ages = ages.replace(/days?/, "天");
      ages = ages.replace(/hours?/, "小时");
      ages = ages.replace(/minutes?/, "分");
      ages = ages.replace(/seconds?/, "秒");
      ages = ages.replace(/\d+/g, '<span style="color:{{ theme.footer.site_runtime.color }}">$&</span>');
      div.innerHTML = `{{ __('footer.site_runtime')}} ${ages}`;
    }
    var div = document.createElement("div");
    var copyright = document.querySelector(".copyright");
    document.querySelector(".footer-inner").insertBefore(div, copyright.nextSibling);
    timer();
    setInterval("timer()",1000)
  </script>
{% endif %}
```
{% endraw %}

在 `themes/next/layout/_layout.njk` 文件 body 标签中添加如下代码：

{% raw %}
```bash
    ...
    {% include '_partials/time.njk' %}
  </body>
</html>
```
{% endraw %}

在`themes/next/_config.yml`文件中添加：

```bash
footer:
  # Specify the year when the site was setup. If not defined, current year will be used.
  since: 2024

  ...

  # Web Site runtime
  site_runtime:
    enable: true
    # Specify the date when the site was setup
    since: 20240130100000
    # color of number
    color: "#1890ff"
```

然后在文件 `themes/next/languages/zh-CN.yml` 中补全对应翻译：

```bash
footer:
  powered: "由 %s 强力驱动"
  total_views: 总访问量
  total_visitors: 总访客量
  site_runtime: "本站已稳定运行"
```

### 页面特效

**浮出爱心：**

在`themes/next/source/js/cursor`目录下新建`love.min.js`文件，添加如下代码：

```javascript
!function(e,t,a){function n(){c(".heart{width: 10px;height: 10px;position: fixed;background: #f00;transform: rotate(45deg);-webkit-transform: rotate(45deg);-moz-transform: rotate(45deg);}.heart:after,.heart:before{content: '';width: inherit;height: inherit;background: inherit;border-radius: 50%;-webkit-border-radius: 50%;-moz-border-radius: 50%;position: fixed;}.heart:after{top: -5px;}.heart:before{left: -5px;}"),o(),r()}function r(){for(var e=0;e<d.length;e++)d[e].alpha<=0?(t.body.removeChild(d[e].el),d.splice(e,1)):(d[e].y--,d[e].scale+=.004,d[e].alpha-=.013,d[e].el.style.cssText="left:"+d[e].x+"px;top:"+d[e].y+"px;opacity:"+d[e].alpha+";transform:scale("+d[e].scale+","+d[e].scale+") rotate(45deg);background:"+d[e].color+";z-index:99999");requestAnimationFrame(r)}function o(){var t="function"==typeof e.onclick&&e.onclick;e.onclick=function(e){t&&t(),i(e)}}function i(e){var a=t.createElement("div");a.className="heart",d.push({el:a,x:e.clientX-5,y:e.clientY-5,scale:1,alpha:1,color:s()}),t.body.appendChild(a)}function c(e){var a=t.createElement("style");a.type="text/css";try{a.appendChild(t.createTextNode(e))}catch(t){a.styleSheet.cssText=e}t.getElementsByTagName("head")[0].appendChild(a)}function s(){return"rgb("+~~(255*Math.random())+","+~~(255*Math.random())+","+~~(255*Math.random())+")"}var d=[];e.requestAnimationFrame=function(){return e.requestAnimationFrame||e.webkitRequestAnimationFrame||e.mozRequestAnimationFrame||e.oRequestAnimationFrame||e.msRequestAnimationFrame||function(e){setTimeout(e,1e3/60)}}(),n()}(window,document);
```

**礼花特效：**

在`themes/next/source/js/cursor`目录下新建`firework.js`文件，添加如下代码：

```javascript
class Circle {
  constructor({ origin, speed, color, angle, context }) {
    this.origin = origin
    this.position = { ...this.origin }
    this.color = color
    this.speed = speed
    this.angle = angle
    this.context = context
    this.renderCount = 0
  }

  draw() {
    this.context.fillStyle = this.color
    this.context.beginPath()
    this.context.arc(this.position.x, this.position.y, 2, 0, Math.PI * 2)
    this.context.fill()
  }

  move() {
    this.position.x = (Math.sin(this.angle) * this.speed) + this.position.x
    this.position.y = (Math.cos(this.angle) * this.speed) + this.position.y + (this.renderCount * 0.3)
    this.renderCount++
  }
}

class Boom {
  constructor ({ origin, context, circleCount = 16, area }) {
    this.origin = origin
    this.context = context
    this.circleCount = circleCount
    this.area = area
    this.stop = false
    this.circles = []
  }

  randomArray(range) {
    const length = range.length
    const randomIndex = Math.floor(length * Math.random())
    return range[randomIndex]
  }

  randomColor() {
    const range = ['8', '9', 'A', 'B', 'C', 'D', 'E', 'F']
    return '#' + this.randomArray(range) + this.randomArray(range) + this.randomArray(range) + this.randomArray(range) + this.randomArray(range) + this.randomArray(range)
  }

  randomRange(start, end) {
    return (end - start) * Math.random() + start
  }

  init() {
    for(let i = 0; i < this.circleCount; i++) {
      const circle = new Circle({
        context: this.context,
        origin: this.origin,
        color: this.randomColor(),
        angle: this.randomRange(Math.PI - 1, Math.PI + 1),
        speed: this.randomRange(1, 6)
      })
      this.circles.push(circle)
    }
  }

  move() {
    this.circles.forEach((circle, index) => {
      if (circle.position.x > this.area.width || circle.position.y > this.area.height) {
        return this.circles.splice(index, 1)
      }
      circle.move()
    })
    if (this.circles.length == 0) {
      this.stop = true
    }
  }

  draw() {
    this.circles.forEach(circle => circle.draw())
  }
}

class CursorSpecialEffects {
  constructor() {
    this.computerCanvas = document.createElement('canvas')
    this.renderCanvas = document.createElement('canvas')

    this.computerContext = this.computerCanvas.getContext('2d')
    this.renderContext = this.renderCanvas.getContext('2d')

    this.globalWidth = window.innerWidth
    this.globalHeight = window.innerHeight

    this.booms = []
    this.running = false
  }

  handleMouseDown(e) {
    const boom = new Boom({
      origin: { x: e.clientX, y: e.clientY },
      context: this.computerContext,
      area: {
        width: this.globalWidth,
        height: this.globalHeight
      }
    })
    boom.init()
    this.booms.push(boom)
    this.running || this.run()
  }

  handlePageHide() {
    this.booms = []
    this.running = false
  }

  init() {
    const style = this.renderCanvas.style
    style.position = 'fixed'
    style.top = style.left = 0
    style.zIndex = '999999999999999999999999999999999999999999'
    style.pointerEvents = 'none'

    style.width = this.renderCanvas.width = this.computerCanvas.width = this.globalWidth
    style.height = this.renderCanvas.height = this.computerCanvas.height = this.globalHeight

    document.body.append(this.renderCanvas)

    window.addEventListener('mousedown', this.handleMouseDown.bind(this))
    window.addEventListener('pagehide', this.handlePageHide.bind(this))
  }

  run() {
    this.running = true
    if (this.booms.length == 0) {
      return this.running = false
    }

    requestAnimationFrame(this.run.bind(this))

    this.computerContext.clearRect(0, 0, this.globalWidth, this.globalHeight)
    this.renderContext.clearRect(0, 0, this.globalWidth, this.globalHeight)

    this.booms.forEach((boom, index) => {
      if (boom.stop) {
        return this.booms.splice(index, 1)
      }
      boom.move()
      boom.draw()
    })
    this.renderContext.drawImage(this.computerCanvas, 0, 0, this.globalWidth, this.globalHeight)
  }
}

const cursorSpecialEffects = new CursorSpecialEffects()
cursorSpecialEffects.init()
```

**爆炸特效：**

在`themes/next/source/js/cursor`目录下新建`explosion.min.js`文件，添加如下代码：

```javascript
"use strict";function updateCoords(e){pointerX=(e.clientX||e.touches[0].clientX)-canvasEl.getBoundingClientRect().left,pointerY=e.clientY||e.touches[0].clientY-canvasEl.getBoundingClientRect().top}function setParticuleDirection(e){var t=anime.random(0,360)*Math.PI/180,a=anime.random(50,180),n=[-1,1][anime.random(0,1)]*a;return{x:e.x+n*Math.cos(t),y:e.y+n*Math.sin(t)}}function createParticule(e,t){var a={};return a.x=e,a.y=t,a.color=colors[anime.random(0,colors.length-1)],a.radius=anime.random(16,32),a.endPos=setParticuleDirection(a),a.draw=function(){ctx.beginPath(),ctx.arc(a.x,a.y,a.radius,0,2*Math.PI,!0),ctx.fillStyle=a.color,ctx.fill()},a}function createCircle(e,t){var a={};return a.x=e,a.y=t,a.color="#F00",a.radius=.1,a.alpha=.5,a.lineWidth=6,a.draw=function(){ctx.globalAlpha=a.alpha,ctx.beginPath(),ctx.arc(a.x,a.y,a.radius,0,2*Math.PI,!0),ctx.lineWidth=a.lineWidth,ctx.strokeStyle=a.color,ctx.stroke(),ctx.globalAlpha=1},a}function renderParticule(e){for(var t=0;t<e.animatables.length;t++)e.animatables[t].target.draw()}function animateParticules(e,t){for(var a=createCircle(e,t),n=[],i=0;i<numberOfParticules;i++)n.push(createParticule(e,t));anime.timeline().add({targets:n,x:function(e){return e.endPos.x},y:function(e){return e.endPos.y},radius:.1,duration:anime.random(1200,1800),easing:"easeOutExpo",update:renderParticule}).add({targets:a,radius:anime.random(80,160),lineWidth:0,alpha:{value:0,easing:"linear",duration:anime.random(600,800)},duration:anime.random(1200,1800),easing:"easeOutExpo",update:renderParticule,offset:0})}function debounce(e,t){var a;return function(){var n=this,i=arguments;clearTimeout(a),a=setTimeout(function(){e.apply(n,i)},t)}}var canvasEl=document.querySelector(".fireworks");if(canvasEl){var ctx=canvasEl.getContext("2d"),numberOfParticules=30,pointerX=0,pointerY=0,tap="mousedown",colors=["#FF1461","#18FF92","#5A87FF","#FBF38C"],setCanvasSize=debounce(function(){canvasEl.width=2*window.innerWidth,canvasEl.height=2*window.innerHeight,canvasEl.style.width=window.innerWidth+"px",canvasEl.style.height=window.innerHeight+"px",canvasEl.getContext("2d").scale(2,2)},500),render=anime({duration:1/0,update:function(){ctx.clearRect(0,0,canvasEl.width,canvasEl.height)}});document.addEventListener(tap,function(e){"sidebar"!==e.target.id&&"toggle-sidebar"!==e.target.id&&"A"!==e.target.nodeName&&"IMG"!==e.target.nodeName&&(render.play(),updateCoords(e),animateParticules(pointerX,pointerY))},!1),setCanvasSize(),window.addEventListener("resize",setCanvasSize,!1)}
```

**浮出文字：**

在`themes/next/source/js/cursor`目录下新建`text.js`文件，添加如下代码：

```javascript
var a_idx = 0;
jQuery(document).ready(function($) {
  $("body").click(function(e) {
    var a = new Array("喜欢我", "不喜欢我");
    var $i = $("<span/>").text(a[a_idx]);
    var x = e.pageX,
    y = e.pageY;
    $i.css({
      "z-index": 99999,
      "top": y - 28,
      "left": x - a[a_idx].length * 8,
      "position": "absolute",
      "color": "#ff7a45"
    });
    $("body").append($i);
    $i.animate({
      "top": y - 180,
      "opacity": 0
    }, 1500, function() {
      $i.remove();
    });
    a_idx = (a_idx + 1) % a.length;
  });
});
```

在`themes/next/layout/_partials`目录下新建一个`page_effects.njk`文件，添加如下代码：

{% raw %}
```bash
{# 鼠标点击特效 #}
{% if theme.cursor_effect == "fireworks" %}
  <script async src="/js/cursor/firework.js"></script>
{% elseif theme.cursor_effect == "explosion" %}
  <canvas class="fireworks" style="position: fixed;left: 0;top: 0;z-index: 1; pointer-events: none;" ></canvas>
  <script src="https://cdn.jsdelivr.net/npm/animejs@2.2.0/anime.min.js"></script>
  <script async src="/js/cursor/explosion.min.js"></script>
{% elseif theme.cursor_effect == "love" %}
  <script async src="/js/cursor/love.min.js"></script>
{% elseif theme.cursor_effect == "text" %}
  <script async src="/js/cursor/text.js"></script>
{% endif %}
```
{% endraw %}

在 `themes/next/layout/_layout.njk` 文件 body 标签中添加如下代码：

{% raw %}
```bash
    ...
    {% include '_partials/time.njk' %}
    {% include '_partials/page_effects.njk' %}
  </body>
</html>
```
{% endraw %}

在`themes/next/_config.yml`文件中添加：

```bash
# mouse click effect: fireworks | explosion | love | text
cursor_effect: love
```

## 参考

http://yifanstar.top/2020/07/19/hexo-blog-creat/

https://hexo-next.readthedocs.io/zh-cn/latest/

https://blog.17lai.site/posts/40300608/

https://zhuanlan.zhihu.com/p/618864711

https://blog.csdn.net/xq151750111/article/details/131101229
