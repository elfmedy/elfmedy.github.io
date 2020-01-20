---
title: 使用 hexo 搭建博客
date: 2019-04-20 09:53:00
tags: [hexo]
---

## 1.基本步骤

可以参考 [https://hexo.io/docs/index.html](https://hexo.io/docs/index.html) 上的官方文档来搭建环境。这里总结一下主要步骤，首先是安装 nodejs 和 npm 包管理器，然后通过包管理器安装 hexo。

<pre class="themepre">
sudo apt install nodejs         <span class="themespan">#安装 nodejs</span>
sudo apt install npm            <span class="themespan">#安装 npm 包管理器</span>
sudo npm install -g hexo-cli    <span class="themespan">#全局安装 Hexo</span>
</pre>

Node.js 是一个基于 Chrome JavaScript V8 引擎的 JavaScript 语言的运行时库。浏览器通常会内置 JavaScript 的解释器，所以 JavaScript 的代码可以被浏览器执行。如果脱离浏览器环境开发，比如需要在本地开发 JavaScript 应用的话，就可以安装和使用 Node.js 这个运行时库。

npm 是一个 Node.js 的包管理器，可以让 JavaScript 的开发者方便的共享和重用代码，有点类似于 ubuntu 的 apt 工具。npm 是随着 Node.js 一起发布的，所以通常来说，安装了 Node.js 也就安装了 npm。用户可以通过 npm install 来安装 Node.js 的包，安装分为全局安装和本地安装，区别是是否带有参数 -g。本地安装会把包安装在 ./node_modules 目录，而全局安装则会安装在 /usr/local 或者 node 的安装目录中。

对于一个新的目录，可以进行如下初始化，其中 npm install 会根据目录中的 package.json 文件安装包以及它们的依赖包。

<pre class="themepre">
$ hexo init <folder>   <span class="themespan"># 在目录中创建 hexo 环境</span>
$ cd <folder>
$ npm install          <span class="themespan"><span class="themespan"># 在目录中建立和填充 node_modules 目录，是 hexo 需要使用的各个支持包</span>
</pre>

初始化完成之后的目录结构如下，其中 source 目录会存放以后写的文章，而 themes 是存放 hexo 主题的目录。

<pre class="themepre">
node_modules\
public\         <span class="themespan"># 这个目录是 hexo g 生成的网站所在的目录，初始的时候这个目录不存在</span>
scaffolds\
source\         <span class="themespan"># .md 文章存放的目录</span>
themes\         <span class="themespan"># 主题模板存放目录</span>
_config.yml
package.json
</pre>

安装完了之后可以安装新的主体模板（参考官网的步骤来安装），以及修改 \_config.yml 文件，完成本地适配工作。至于写文章，主要步骤如下

- 使用 hexo new "文章名字" 来新建新的文档，也可以在 sources/\_posts/ 目录下直接新建 .md 文章
- 使用 hexo g 读取 source 目录下的用户文章，生成 public 目录的内容，这个也是网页最终呈现的内容
- 使用 hexo s 可以在本地预览站点，站点的根目录也就是 public 目录。浏览器会读取 public 目录中的内容并显示

## 2.github pages 相关

上一节说到了如何在本地环境下创建 hexo 环境、利用 hexo 写文章、生成网站并且进行本地预览。不过我们希望把网站放到公网上给别人访问，github pages 可以帮助我们达到这个目的。github pages 是 github 公司提供的免费的静态网页托管服务，它对应于用户的一个特殊的仓库，这个仓库的名字必须是 username.github.io，其中 username 是用户的 github 用户名。

我们需要把 hexo public 目录中的内容上传到仓库的 master 分支（github pages 只会呈现 master 分支的内容）。当然为了方便在不同的地方写文章，最好也把 hexo 的开发环境也保存在 github 中，这样换一个环境也不需要完全重新安装各种依赖包。具体做法是把 public 目录放在 master 分支中，而 hexo 开发目录放在分支中，比如 develop 分支中。

develop 分支中的目录结构如下，因为对 node_modules 中的模块做了一些个人的修改，所以我这里把 node_nodule 整个目录都放到 git 中。

<pre class="themepre">
node_modules\	<span class="themespan"># 目录中有一些后面关于语法高亮的修改，所以这里把这个目录也放到 git 中</span>
scaffolds\
source\   		<span class="themespan"># source 目录中存放了 .md 后缀的文章</span>
theme\
\_config.yml
package.json
package-lock.json
</pre>

master 分支的目录结构如下，也就是通过 hexo s 生成的 public 目录中的内容

<pre class="themepre">
about\
archives\
css\
images\  
index.html  
js\
...
</pre>

通常我选择 clone 仓库到两个不同的目录，分别切换为 master 分支和 develop 分支。在 develop 分支进行文章的撰写并且生成 public 目录，然后通过目录比较工具（比如 beyond compare）或者直接拷贝 public 目录中的内容到 master 分支，再上传 master 分支就行了。

## 3.环境迁移

如果需要在一台新的电脑上配置开发环境，可以按照以下步骤进行，首先就是同样需要在新的电脑上安装 hexo 的支持环境

<pre class="themepre">
sudo apt install nodejs         <span class="themespan">#安装 nodejs</span>
sudo apt install npm            <span class="themespan">#安装 npm 包管理器</span>
sudo npm install -g hexo-cli    <span class="themespan">#全局安装 Hexo</span>
</pre>

然后可以把 github 上面的代码 clone 下来。在上一节中，为了方便迁移开发环境，我们把整个开发的目录放在了 develop 分支中，所以这里 clone 仓库并且切换到 develop 分支就可以进行写作了。

<pre class="themepre">
git clone username.github.io  <span class="themespan">#clone 用户自己的 github pages 仓库，注意把 username 替换成自己的仓库名</span>
git checkout develop      <span class="themespan">#切换到 develop 分支</span>
hexo g                    <span class="themespan">#生成 public 目录</span>
hexo s                    <span class="themespan">#运行本地服务器</span>
</pre>

运行 hexo g 时可能会有如下错误，也就是 hexo-server 加载失败。hexo-server 是使用本地浏览器访问的时候需要用到的包。

<pre class="themepre">
$ hexo g
ERROR Plugin load failed: hexo-server
Error: Cannot find module './db.json'
    at Function.Module._resolveFilename (internal/modules/cjs/loader.js:668:15)
    at Function.Module._load (internal/modules/cjs/loader.js:591:27)
    at Module.require (internal/modules/cjs/loader.js:723:19)
    at require (internal/modules/cjs/helpers.js:14:16)
</pre>

此时可以运行 npm install hexo-server 来重新安装 server 环境。

## 3.图片显示

图片显示是 hexo 比较麻烦的地方。一种方法是使用外部链接，比如把图片放在支持图片外链云盘中，文章中使用云盘中的图片链接。另外一种办法是将图片也保存在 hexo 站点中，默认情况下只能将图片放到主题的 images 目录（想新建目录来专门保存图片是不行的，hexo g 生成的网站的时候只会拷贝特定目录中的内容到）。不过 hexo 提供了 post_asset_folder 选项，这个选项可以在创建文章的时候创建一个和文章名字同名的文件夹，图片可以放在这个目录中。

<pre class="themepre">
post_asset_folder: true		<span class="themespan"># 在 _config.yml 目录中</span> 
</pre>

具体如果需要在文章中引用图片，可以使用下面的语法

<pre class="themepre">
{% raw %}{% asset_img "image" "image desc" %}{% endraw %}           <span class="themespan">// 显示图片</span>
{% raw %}{% asset_img "image" xxx xxx "image desc" %}{% endraw %}   <span class="themespan">// 显示图片，设置图片的宽高参数，可以只设置宽，那么高是自动计算的</span>
</pre>

## 4.语法高亮

hexo 使用的高亮插件不支持对代码中的特殊语句进行高亮，比如

<pre class="themepre">
code  // 注释1
code  // 注释2
</pre>

想对其中的 注释1 单独高亮是不支持的。我的想法是直接插入 html 代码块，在特定的需要高亮的地方有插入 span 标签，然后在 css 中设置格式。这样做有一个不好的点就是关键字高亮比较麻烦，不过这个也不是我的需求。一段插入的 html 代码如下所示，可以设置 pre.themepre 和 span.themespan 的 css 属性，让 html 段呈现比较好的效果。

<pre class="themepre" class="undo">
&lt;pre class=&quot;themepre&quot;&gt;
code  &lt;span class=&quot;themespan&quot;&gt;// 注释1&lt;/span&gt;
code  // 注释2
&lt;/pre&gt;
</pre>

采用上面直接插入 html 块的方式显示代码，会发现一些代码出现乱码或者显示错误，究其原因就是 marked 处理不会对 html 标签中的任何内容做处理，而由浏览器自行解析，比如下面的一段代码

<pre class="themepre" class="undo">
&amp;para_start_addr_1; 
&lt;code&gt;&lt;/code&gt;
</pre>

浏览器解析的结果如下，一些 html 的标记被浏览器“正确的”解析了（其中的 &amp;para_start_addr_1，虽然不符合 &amp;para; 但是浏览器还是变成了特殊符号）。

<pre class="themepre" class="undo">
&para_start_addr_1; 
<code></code><span class="themespan">//此处为空行，因为浏览器解析 &lt;code&gt; 为标签</span>
</pre>

hexo 处理代码的时候是直接调用的 node_modules/hexo-renderer-marked 这个模块进行处理的，而其中的 renderer.js 又是调用的 node_modules/marked 进行的解析。对 node_modules/hexo-renderer-marked/lib/renderer.js 中的部分代码做修改，添加对于 html block 段的特殊处理，只保留用于高亮的特殊 html 标记，其他的 html 标记全部做转换。这段修改是根据 html 段以 &lt;pre class=&quot;themepre&quot;&gt; 开头来做匹配的，如果仍然想 html 块中的内容被按照 html 来解析，只需要让 html 段不符合匹配格式即可，比如修改 html 段为以 &lt;pre class=&quot;themepre&quot; class=&quot;undo&quot;&gt; 开头。

这段代码只是做了简单处理，所以还是会有一些情况，比如代码中有 &lt;/span&gt; 会显示不了，不过关系不是太大。

<pre class="themepre" class="undo">
<span class="themespan">// node_modules/hexo-renderer-marked/lib/renderer.js</span>
Renderer.prototype.html = function(text) {
  var result;
  <span class="themespan">// 这里以 &lt;pre class=&quot;themepre&quot;&gt; 开头作为匹配条件</span>
  <span class="themespan">// 如果不想使用这段配置的函数，也就是仍然想其中的内容被作为 html 解析，可以简单的对 html 标签做修改，比如改为 &lt;pre class=&quot;themepre&quot; class=&quot;undo&quot;&gt;</span>
  if (/^&lt;pre class=&quot;themepre&quot;&gt;/.test(text)) {
    <span class="themespan">// 首先替换掉 &amp, &lt, &gt 等特殊符号</span>
    text = text.replace(/&amp;/g, &quot;&amp;amp;&quot;).replace(/&lt;/g, &#39;&amp;lt;&#39;).replace(/&gt;/g, &#39;&amp;gt;&#39;).replace(/&quot;/g, &#39;&amp;quot;&#39;).replace(/&#39;/g, &#39;&amp;#39;&#39;);

    <span class="themespan">// 对于两个高亮的 html 语句，再替换回来</span>
    text = text.replace(/&amp;lt;pre class=&amp;quot;themepre&amp;quot;&amp;gt;/g, &quot;&lt;pre class=\&quot;themepre\&quot;>&quot;);
    text = text.replace(/&amp;lt;\/pre&amp;gt;/g, &quot;&lt;/pre&gt;&quot;);

    text = text.replace(/&amp;lt;span class=&amp;quot;themespan&amp;quot;&amp;gt;/g, &quot;&lt;span class=\&quot;themespan\&quot;&gt;");
    text = text.replace(/&amp;lt;\/span&amp;gt;/g, &quot;&lt;/span&gt;&quot;);

    result = text;
  } else {
    result = text;
  }

  return result;
};
</pre>