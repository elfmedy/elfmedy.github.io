---
title: 使用 hexo 搭建博客
date: 2019-04-20 09:53:00
tags: [hexo]
---

## 1.基本步骤

可以参考 [https://hexo.io/docs/index.html](https://hexo.io/docs/index.html) 官方文档来搭建环境。主要步骤为

<pre class="themepre">
$ hexo init <folder>   <span class="themespan"># 在目录中创建开发环境</span>
$ cd <folder>
$ npm install          <span class="themespan"><span class="themespan"># 在目录中建立和填充 node_modules 目录，是 hexo 的各个支持库</span>
</pre>

完成之后的目录结构如下

<pre class="themepre">
node_modules\
public\         <span class="themespan"># 这个目录是 hexo g 生成的网站所在的目录，初始的时候这个目录不存在</span>
scaffolds\
source\         <span class="themespan"># .md 文章存放的目录</span>
themes\         <span class="themespan"># 主题模板存放目录</span>
_config.yml
package.json
</pre>

安装完了之后可以安装新的主体模板（参考官网），以及修改 \_config.yml 文件，完成本地适配工作

- 使用 hexo new "文章名字" 来新建新的文档，也可以在 sources/\_posts/ 目录下直接新建 .md 文章
- 使用 hexo g 生成 public 目录的内容，这个是网站的内容
- 使用 hexo s 可以在本地预览站点，站点的根目录也就是 public 目录

## 2.github pages 相关

github pages 提供静态网站的支持，所以其实就是把 hexo public 中的内容全部上传就可以了。当然最好也把 hexo 的开发环境也保存在 github 中，这样也记录了对于主题的一些修改。做法是把 public 目录放在 master 分支中，而 hexo 开发目录放在分支中，比如 develop 分支中。

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

master 分支的目录结构如下，就是访问到的个人博客的目录

<pre class="themepre">
about\
archives\
css\
images\  
index.html  
js\
...
</pre>

## 3.图片显示

图片显示是 hexo 比较麻烦的地方。一种方法是使用外部链接，比如把图片放在云盘中，文章中使用云盘中的图片链接。另外一种办法是将图片也保存在 hexo 站点中，默认情况下只能将图片放到主题的 images 目录（想新建目录来专门保存图片是不行的，hexo g 生成的网站的时候只会拷贝特定目录中的内容到）。不过 hexo 提供了 post_asset_folder 选项，这个选项可以在创建文章的时候创建一个和文章名字同名的文件夹，图片可以放在这个目录中。

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