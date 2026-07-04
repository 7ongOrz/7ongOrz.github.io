# Hexo Migration Handoff Prompt

这个文档用于在迁移项目路径、开启新对话或交给新的 AI 助手时无缝接上当前 Hexo 博客迁移工作。新助手应先阅读本文件，再查看仓库。

## 复制给新对话的提示词

```text
你现在接手的是我的 Hexo 博客源站仓库，请先完整阅读本仓库的 docs/hexo-migration-handoff-prompt.md，再继续工作。

项目背景：
- 当前站点是 https://blog.tongorz.top。
- GitHub 仓库是 7ongOrz/7ongOrz.github.io，当前主分支使用 master。
- 这个仓库现在是 Hexo 源站仓库，不再提交 Hexo 生成后的 public/ 或旧的 article/、archives/、tags/ 静态产物。
- GitHub Pages 发布方式是 GitHub Actions：每次推送 master 后在 CI 里安装依赖、运行 Hexo 生成 public/，再把 public/ 作为 Pages artifact 发布。
- Pages 设置应为 Settings -> Pages -> Build and deployment -> Source: GitHub Actions。
- 当前不要随意改 hexo-theme-next 主题源码；主题本体来自 npm 依赖 hexo-theme-next。所有自定义尽量放在 _config.next.yml、source/_data/*.njk、source/_data/*.styl 和文章 Markdown 中。
- 我曾经从旧 docker/app 目录迁移出来很多文章和旧 NexT 自定义。现在迁移目标是保留功能，同时让主题继续可以同步上游 npm 更新。

关键约束：
- 不要在本地随意安装 npm 依赖，除非我明确同意；之前这个机器没有 npm，本地主要做静态分析和线上验证。
- 不要制造或遗留 node_modules、public、.cache、.hexo、.deploy_git、.next 等构建/缓存目录。
- 不要写 /private/tmp 或 /var/folders 之类临时目录；如果需要 HTTP 请求，用 Ruby Net::HTTP 或浏览器工具，不要用 Ruby open-uri，因为 open-uri 会尝试创建 open-uri 临时文件。
- 手动改文件用 apply_patch；不要用 cat/heredoc 直接写文件。
- 不要回退用户或其他工具已经做过的修改。
- 需要 git add/commit/push 时，可能要申请提升权限，因为 sandbox 可能无法写 .git/index.lock。

仓库当前重要文件：
- package.json：Hexo 和插件依赖。当前 Hexo 为 8.1.2，hexo-theme-next 为 8.28.0，hexo-filter-mathjax 为 0.11.0。
- _config.yml：Hexo 站点配置，包括 url、permalink、文章资源目录、服务端 MathJax、搜索、Giscus 评论等。
- _config.next.yml：NexT 外置主题配置。这里启用了 custom_file_path.style: source/_data/styles.styl 和 bodyEnd: source/_data/body-end.njk；NexT 前端 MathJax/Katex 渲染保持关闭，避免与 hexo-filter-mathjax 重复渲染。
- source/_posts：文章 Markdown 和每篇文章自己的图片资源目录。
- source/_data/styles.styl：NexT 自定义样式。当前包含 MathJax 和移动端正文溢出兜底。
- source/_data/body-end.njk：NexT body 末尾注入。当前包含 canvas-nest、站点运行时间、可选鼠标点击特效，以及为避免 CSS 缓存导致手机端溢出的内联移动端兜底样式。
- source/images/81060761_p0.jpg：侧栏头像。
- source/js/cursor：旧站迁移过来的鼠标点击特效脚本。
- source/CNAME：自定义域名 blog.tongorz.top。
- .github/workflows/pages.yml：GitHub Pages 构建发布 workflow，只对 master 发布。
- .github/dependabot.yml：检查 npm 和 GitHub Actions 更新。
- README.md：当前仓库目录逻辑和常用命令说明。

当前发布和依赖策略：
- workflow 使用 Node.js 22。
- workflow 先 apt-get install pandoc，再 npm install --no-audit --no-fund，再 npm run clean 和 npm run build。
- 当前没有提交 package-lock.json；如果以后在有 npm 的环境中确认构建成功，建议生成并提交 package-lock.json，然后把 workflow 的 npm install 改为 npm ci --no-audit --no-fund。
- 当前依赖基线：hexo 8.1.2、hexo-theme-next 8.28.0、hexo-filter-mathjax 0.11.0、hexo-next-giscus 1.3.0 等。

评论方案：
- NexT 内置的 disqus/gitalk/utterances 等在 _config.next.yml 中未启用。
- 实际评论使用 package.json 里的 hexo-next-giscus 插件，并在 _config.yml 的 giscus 配置中启用。
- giscus repo 为 7ongOrz/comments，之前通过 GitHub API 验证过 repo 存在、public、has_discussions=true，repo node_id 与 _config.yml 中 repo_id 匹配。

域名和 URL：
- source/CNAME 内容是 blog.tongorz.top。
- _config.yml 中 url 是 https://blog.tongorz.top。
- permalink 是 article/:year:month:day:abbrlink.html。
- 注意：由于时区/date 解析，实际线上文章 URL 不要只靠源码推导，优先从 https://blog.tongorz.top/search.xml 读取真实 URL。

最近已经完成的关键提交：
- 3aa11cd chore: migrate Hexo site source
  - 把仓库迁成 Hexo 源站结构，移除旧生成产物，保留文章、图片、配置、自定义脚本、GitHub Actions。
- cba8fb6 fix: improve MTGAN post layout
  - 修复《基于多任务生成对抗网络的高光谱图像分类》文章排版和图片。
- 65c7fd1 fix: clean up post math layout
  - 修复 HyperViTGAN 和《基于半监督卷积生成对抗网络的高光谱图像分类》的数学公式/排版。
- f3ec0e8 docs: note current NexT setup
  - 给旧的 Hexo/NexT 自用配置文档添加当前迁移说明。
- 97c23ea chore: align Pages workflow
  - Pages workflow 只使用 master；.gitignore 忽略缓存/构建目录；README 说明 master 发布。
- 45f6fd8 fix: improve mobile math layout
  - 启用 source/_data/styles.styl，给块级 MathJax 加移动端滚动兜底；初步拆分 MTGAN 长公式。
- b9fe044 fix: wrap MTGAN equations
  - 进一步把 MTGAN 文章中公式 (7)-(11) 改成多行 aligned，手机端不再撑宽整页。
- 01eb81e fix: contain mobile canvas background
  - 初步给 canvas-nest 背景 canvas 加宽度约束。
- 9b3257f fix: inline canvas sizing guard
  - 把 canvas-nest 的宽度/触摸事件兜底移到 body-end.njk 内联样式里，避免 main.css 缓存影响。
- 2cba498 fix: guard mobile post overflow
  - 添加移动端正文溢出兜底，长 URL、长英文/符号串、行内公式、版权链接不再撑开整页。
- be8a2e4 Merge pull request #1
  - Dependabot 更新 actions/checkout 到 v7。
- c5f5d13 Merge pull request #2
  - Dependabot 更新 hexo-theme-next 到 8.28.0。
- 2dadba2 Merge pull request #3
  - Dependabot 更新 hexo-filter-mathjax 到 0.11.0，服务端 MathJax 升到 MathJax 4。

当前 HEAD：
- 继续工作前先运行 `git status --short --branch` 和 `git log --oneline --decorate -5`，以当前仓库状态为准。
- 最近的依赖更新基线是 2dadba2，后续本文件可能再随维护提交前进。

已经验证过的内容：
- git status 干净：## master...origin/master。
- 没有生成 node_modules、public、.cache、.hexo、.deploy_git、.next。
- GitHub Pages 构建成功。
- 线上 36 篇文章真实 URL 来源于 search.xml，使用 375px 手机视口批量检查 rootScrollWidth/bodyScrollWidth，最终 36/36 篇文章不再出现整页横向溢出。
- 单独复测过曾经异常的文章 /article/202304051f241dff.html，最终 rootScrollWidth=375、bodyScrollWidth=375、postBody scrollWidth=clientWidth=335。
- 单独复测过导航曾超时的一篇 /article/202303156ca8c96f.html，最终 rootScrollWidth=375、bodyScrollWidth=375。
- MTGAN 文章 /article/202406255854198f.html 修复后 375px 手机视口下 rootScrollWidth=375、bodyScrollWidth=375。

后续如果继续 review，应按以下顺序：
1. 先运行 git status --short --branch，确认是否有未提交改动。
2. 查看最近提交和本文件，避免重复修复。
3. 不要编辑主题源码；优先通过 _config.next.yml、source/_data/styles.styl、source/_data/body-end.njk 或文章 Markdown 修。
4. 对 Markdown 文章做静态检查：frontmatter、<!--more-->、图片资源、链接、代码围栏、公式分隔符。
5. 对移动端布局问题，必须用真实线上 URL 或本地构建后的页面在 375px 左右视口验证 rootScrollWidth/bodyScrollWidth。
6. 推送后等待 GitHub Actions Pages 成功，再复测线上。
7. 完成后确认 git status 干净，并确认未生成缓存/构建目录。
```

## 当前仓库结构和职责

当前仓库是 Hexo 源站仓库，而不是生成后的静态站仓库。

核心结构：

- `_config.yml`：Hexo 通用站点配置。
- `_config.next.yml`：NexT 主题配置，采用 alternate theme config 模式。
- `source/_posts/`：Markdown 文章和每篇文章对应的图片资源目录。
- `source/_data/`：NexT 官方支持的自定义注入文件。
- `source/images/`：站点级图片，如头像。
- `source/js/cursor/`：旧站迁移来的鼠标点击特效脚本。
- `.github/workflows/pages.yml`：GitHub Actions 构建并发布 GitHub Pages artifact。
- `.github/dependabot.yml`：依赖更新检查。
- `README.md`：项目说明。

不要把 `public/`、`node_modules/`、`.deploy_git/` 等构建产物提交进仓库。

## 已做过的迁移决策

### 源码和发布产物分离

现在仓库只保留 Hexo 源码和配置。GitHub Actions 每次构建出 `public/`，然后作为 Pages artifact 发布。这样仓库不会同时维护源码和生成 HTML，后续更标准、冲突更少。

### 使用 master 分支

用户明确不想维护多余分支，所以 workflow、README 和远程发布都统一到 `master`。不要无故新建 `main` 或其他部署分支。

### 主题升级策略

主题本体由 npm 依赖 `hexo-theme-next` 管理。不要直接修改主题包源码。保留自定义功能的方式是：

- `_config.next.yml`：NexT 配置。
- `source/_data/styles.styl`：自定义样式。
- `source/_data/body-end.njk`：body 末尾注入。
- `source/js/cursor/*`：自定义鼠标特效脚本。
- 文章 Markdown：具体文章内容和公式。

这样以后更新 `hexo-theme-next` 版本时，不会因为改过主题源码而难以同步上游。

### 评论方案

实际评论方案是 `hexo-next-giscus`，配置在 `_config.yml` 的 `giscus:` 下。

之前验证过：

- `7ongOrz/comments` 仓库存在。
- 仓库是 public。
- Discussions 已开启。
- `_config.yml` 中 `repo_id` 与 GitHub API 返回的 node_id 匹配。

### MathJax 和移动端

发现过 MTGAN 文章手机端宽度被 MathJax SVG 长公式撑到 566px。修复方式：

- 公式由 `hexo-filter-mathjax` 在服务端渲染；`_config.next.yml` 中 NexT 前端 `math.mathjax.enable` 和 `math.katex.enable` 均保持 `false`，避免重复加载/重复渲染。
- `hexo-filter-mathjax` 0.11.0 使用 MathJax 4，`_config.yml` 的 `mathjax.tags` 使用 `ams`，由插件加载 AMS TeX 包，以支持现有文章里的 `align`、`aligned`、`cases`、`bmatrix` 和 `\tag`。
- 文章源码尽量使用标准 TeX 命令；不要再用 `\scr` 或 `\textless` 这类在 MathJax 4 服务端渲染中不稳定的写法。
- 在 `source/_data/styles.styl` 中给 `.post-body mjx-container[display='true']` 添加 `max-width: 100%`、`min-width: 0 !important`、`overflow-x: auto`、`width: 100%`。
- 将 MTGAN 文章中公式 `(7)` 到 `(11)` 拆成多行 `aligned`。
- 对行内公式和长文本添加移动端兜底，避免长 URL、长英文/符号串撑开正文。

### canvas-nest 移动端问题

`canvas-nest.js` 会动态插入 `body > canvas#c_n...`。在手机浏览器里它曾使用 `window.innerWidth=390px`，但文档可用宽度是 375px，导致页面横向多出 7px。

修复方式：

- 在 `source/_data/body-end.njk` 内联样式里添加：
  - `body > canvas[id^='c_n'] { max-width: 100% !important; width: 100% !important; pointer-events: none !important; }`
- 同时在 `source/_data/styles.styl` 中保留长期样式兜底。

## 重要验证命令

以下命令都应在仓库根目录运行：

```sh
git status --short --branch
git log --oneline --decorate -12
git diff --check
find . -maxdepth 3 -type d \( -name node_modules -o -name public -o -name .cache -o -name .hexo -o -name .deploy_git -o -name .next \)
```

YAML 配置检查：

```sh
ruby -ryaml -e 'YAML.safe_load(File.read("_config.next.yml"), aliases: true); YAML.safe_load(File.read("_config.yml"), aliases: true); puts "YAML_OK"'
```

检查 NexT 自定义样式路径是否存在：

```sh
ruby -e 'config = File.read("_config.next.yml"); path = config[/^\s*style:\s*(\S+)/, 1]; puts "style=#{path}"; puts File.exist?(path) ? "STYLE_EXISTS" : "STYLE_MISSING"'
```

检查 Markdown 代码围栏：

```sh
ruby -e 'Dir["source/**/*.md"].sort.each do |path|; fence = false; count = 0; File.readlines(path).each do |line|; if line.start_with?("```") || line.start_with?("~~~"); fence = !fence; count += 1; end; end; puts "UNCLOSED_FENCE #{path} toggles=#{count}" if fence; end'
```

检查块级数学分隔符是否成对：

```sh
ruby -e 'Dir["source/_posts/*.md"].sort.each do |path|; text = File.read(path); display = text.scan(/(?<!\\)\$\$/).size; puts "ODD_DISPLAY_DOLLAR #{path} count=#{display}" if display.odd?; lbr = text.scan(/(?<!\\)\\\[/).size; rbr = text.scan(/(?<!\\)\\\]/).size; puts "MISMATCH_BRACKET_MATH #{path} open=#{lbr} close=#{rbr}" if lbr != rbr; lpar = text.scan(/(?<!\\)\\\(/).size; rpar = text.scan(/(?<!\\)\\\)/).size; puts "MISMATCH_PAREN_MATH #{path} open=#{lpar} close=#{rpar}" if lpar != rpar; end'
```

检查图片资源引用。注意 Markdown 尖括号路径如 `](<path with spaces>)` 是合法的，脚本要剥掉尖括号：

```sh
ruby -e 'root = Dir.pwd; errors = []; Dir["source/**/*.md"].sort.each do |path|; text = File.read(path); refs = []; text.scan(/!\[[^\]]*\]\(([^)]+)\)/) { |m| refs << m[0] }; text.scan(/<img\b[^>]*\bsrc=["\x27]([^"\x27]+)["\x27]/i) { |m| refs << m[0] }; refs.each do |ref|; next if ref =~ /\A(?:https?:)?\/\//i || ref.start_with?("#", "mailto:", "data:"); clean = ref.strip; clean = clean[1...-1] if clean.start_with?("<") && clean.end_with?(">"); clean = clean.split(/[?#]/, 2).first; next if clean.empty?; candidates = []; candidates << File.expand_path(clean.sub(%r{\A/}, "source/"), root) if clean.start_with?("/"); candidates << File.expand_path(clean, File.dirname(File.expand_path(path, root))); candidates << File.expand_path(File.join("source", clean), root); errors << "MISSING_ASSET #{path} -> #{ref}" unless candidates.any? { |c| File.exist?(c) }; end; end; puts errors'
```

检查每篇文章 frontmatter、`abbrlink` 和 `<!--more-->`：

```sh
ruby -ryaml -rdate -e 'seen = Hash.new { |h,k| h[k]=[] }; Dir["source/_posts/*.md"].sort.each do |path|; text = File.read(path); unless text.start_with?("---\n"); puts "NO_FRONTMATTER #{path}"; next; end; parts = text.split(/^---\s*$/, 3); fm = YAML.safe_load(parts[1], permitted_classes: [Date, Time], aliases: true) || {}; %w[title date abbrlink categories tags].each { |k| puts "MISSING_FM #{path} #{k}" unless fm.key?(k) }; seen[fm["abbrlink"].to_s] << path if fm["abbrlink"]; puts "NO_MORE #{path}" unless text.include?("<!--more-->"); puts "MULTI_MORE #{path}" if text.scan(/<!--more-->/).size > 1; end; seen.each { |abbr, paths| puts "DUP_ABBRLINK #{abbr} #{paths.join(" | ")}" if paths.size > 1 }'
```

检查 GitHub Pages 最近一次 workflow：

```sh
ruby -rnet/http -rjson -ruri -e 'uri=URI("https://api.github.com/repos/7ongOrz/7ongOrz.github.io/actions/runs?branch=master&per_page=1"); res=Net::HTTP.get_response(uri); data=JSON.parse(res.body); run=data.fetch("workflow_runs").first; puts [run["id"], run["head_sha"][0,7], run["name"], run["status"], run["conclusion"], run["html_url"]].join(" | ")'
```

等待指定提交的 Pages workflow 完成：

```sh
ruby -rnet/http -rjson -ruri -e '$stdout.sync=true; target="COMMIT7"; uri=URI("https://api.github.com/repos/7ongOrz/7ongOrz.github.io/actions/runs?branch=master&per_page=5"); 30.times do; res=Net::HTTP.get_response(uri); data=JSON.parse(res.body); run=data.fetch("workflow_runs").find { |r| r["head_sha"].start_with?(target) }; if run; puts [run["id"], run["head_sha"][0,7], run["status"], run["conclusion"], run["html_url"]].join(" | "); break if run["status"] == "completed"; end; sleep 10; end'
```

## 移动端线上批量验证方法

之前最终验证使用的是浏览器工具，在 390x844 viewport 下页面实际 `documentElement.clientWidth` 为 375。判断标准：

- `document.documentElement.scrollWidth <= document.documentElement.clientWidth + 2`
- `document.body.scrollWidth <= document.documentElement.clientWidth + 2`

真实文章 URL 从 `search.xml` 获取，不要只从 frontmatter 推导：

```sh
ruby -rnet/http -ruri -e 'xml = Net::HTTP.get(URI("https://blog.tongorz.top/search.xml")); xml.scan(/<url><!\[CDATA\[(.*?)\]\]><\/url>|<url>(.*?)<\/url>/).flatten.compact.each { |u| puts u if u.include?("/article/") }'
```

浏览器批量验证时注意：

- 页面内 `evaluate` 环境不支持 `setTimeout`，延迟要在浏览器外层 JS 里做。
- `data:` URL 被浏览器安全策略禁止，不要绕过。
- 验证后调用 viewport reset。
- 给 URL 加查询参数如 `?scan=COMMIT7` 可以绕过 HTML 缓存；但 `main.css` 有 `Cache-Control: max-age=600`，所以关键移动端兜底同时放入了 `body-end.njk` 内联样式，避免 CSS 缓存影响复测。

浏览器批量验证核心逻辑示例：

```js
const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms));
const viewport = await browser.capabilities.get('viewport');
await viewport.set({ width: 390, height: 844 });

const articlePaths = [
  // 从 search.xml 得到的 /article/*.html 路径
];

const results = [];
for (const path of articlePaths) {
  await tab.goto(`https://blog.tongorz.top${path}?scan=COMMIT7`);
  await delay(650);
  const result = await tab.playwright.evaluate(() => {
    const root = document.documentElement;
    const body = document.body;
    const vw = root.clientWidth;
    const postBody = document.querySelector('.post-body');
    const mathOverflowCount = [...document.querySelectorAll('mjx-container[display="true"]')].filter(el => {
      const r = el.getBoundingClientRect();
      return r.right > vw + 2 || r.width > vw + 2;
    }).length;
    return {
      title: document.title.replace(" | AhTong's blog", ""),
      vw,
      rootScrollWidth: root.scrollWidth,
      bodyScrollWidth: body.scrollWidth,
      postBodyScrollWidth: postBody ? postBody.scrollWidth : null,
      postBodyClientWidth: postBody ? postBody.clientWidth : null,
      mathCount: document.querySelectorAll('mjx-container[display="true"]').length,
      mathOverflowCount
    };
  });
  results.push({
    path,
    ok: result.rootScrollWidth <= result.vw + 2 && result.bodyScrollWidth <= result.vw + 2,
    ...result
  });
}

console.log(JSON.stringify({
  total: results.length,
  ok: results.filter(x => x.ok).length,
  bad: results.filter(x => x.ok === false).length,
  badItems: results.filter(x => x.ok === false)
}, null, 2));

await viewport.reset();
```

## 已知最终线上文章 URL 数量

最终从 `search.xml` 扫到 36 篇文章。最终批量验证结果：

- total: 36
- ok: 36
- bad: 0

其中曾单独关注过：

- `/article/202406255854198f.html`：MTGAN 文章，MathJax 长公式问题已修复。
- `/article/202304051f241dff.html`：曾有 canvas-nest 和长文本轻微溢出，已修复。
- `/article/202303156ca8c96f.html`：批量导航曾超时，单独复测通过。

## 继续工作的原则

1. 优先保留现有结构，不做无关重构。
2. 不要改主题源码，避免影响后续 `hexo-theme-next` 上游更新。
3. 主题相关自定义走 `_config.next.yml` 和 `source/_data`。
4. 文章问题优先改文章 Markdown；全站排版问题优先加小范围自定义样式。
5. 每次推送后都等 Pages workflow 成功，再验证线上效果。
6. 如果用户要换主题，需要重新适配 `_config.next.yml` 和 `source/_data` 里的 NexT 专属配置；`source/_posts` 文章和图片大多可以保留。
7. 如果用户想摆脱 npm，可评估 Hugo；如果新建现代内容站，可评估 Astro；但当前博客继续 Hexo + NexT 是合理且稳定的。
