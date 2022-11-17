---
title: "org mode写博客"
description: "使用org mode和ox-hugo写博客."
date: 2022-11-17T00:00:00+08:00
lastmod: 2022-11-17T20:00:09+08:00
tags: ["emacs"]
categories: ["emacs"]
draft: false
author: "Narcissus"
---

org基本和markdown类似，但是更加强大，配合emacs使用更加优雅，因此考虑将写博客的流程切换到org,下面将介绍org mode写博客的具体操作。


## 写作方式选择 {#写作方式选择}

org mode写博客需要配合[ox-hugo](https://ox-hugo.scripter.co), Spacemacs只需要在org layers中添加如下变量即可:

```lisp
(org :variables
     org-enable-hugo-support t)
```

ox-hugo有两种博客组织方式:

1.  一个子结构对应一篇博客
2.  一个org文件对应一篇博客

第一种方式的优点是不需要针对每一个org文件管理一些参数设置，同一个org文件设置的参数针对文件中所有博客都有效。因此推荐使用第一种博客组织方式，但混合第二种，博客内容过多则采用多个org文件。第一种方式文章YAML元信息中的title属性值就是一级子结构名称。


## 配置参数 {#配置参数}

写作之前需要配置一些参数，就可以将ord文件正确转换为md文件并存储到正确的hugo框架中存放博客的位置。

```org
#+hugo_base_dir: ~/narcissusBlog/
#+hugo_section: ./posts/
#+hugo_front_matter_format: yaml
#+options: author:nil
#+hugo_auto_set_lastmod: t
```

配置说明：

-   \#+hugo_base_dir表示博客根目录
-   \#+hugo_section表示生成md文件位置，如上配置表示位置为博客根目录下/content/posts/
-   \#+hugo_front_matter_format表示md文件front matter嵌入类型，默认是TOML，可以配置为YAML
-   \#+options: author:nil表示不自动添加作者到YAML中, **这样配置是因为默认添加的是列表，会导致生成md文件无法导出成博客，只能单独每一篇博客手动添加author** 。
-   \#+hugo_auto_set_lastmod设置为真，将会自动添加lastmod元信息到YAML中

每一个一级树下面就是一篇博客， 需要为每一篇博客配置单独的内容，比如导出文件名，博客的YAML等内容，示例代码如下:

```org
:PROPERTIES:
:EXPORT_FILE_NAME: [file_name]
:EXPORT_DESCRIPTION: 使用org mode和ox-hugo写博客.
:EXPORT_HUGO_CUSTOM_FRONT_MATTER: :key value
:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :key '(value1, "value2")
:END:
```

制定了导出文件名、博客描述和front-matter参数，添加多个后面会多一个+符号。

md文档YAML的元信息中包含date属性，这是来自于标题变成DONE状态后的CLOSED时间。但因为只需要日期而不需要时间，可以在org layers中设置如下变量为nil来实现:

```emacs-lisp
(org :variables
     org-log-done-with-time nil)
```


## 导出为md文件 {#导出为md文件}

导出指定subTree结构为md文件，光标定位到具体位置，使用快捷键 **, e e H H**, 当然也可以导出当前org文件的所有博客等选项，会有提示按键。


## 标签与类别 {#标签与类别}

当给博客的标题设置了标签属性后，导出的YAML元信息中将包含标签属性。或者使用 **EXPORT_HUGO_TAGS** 属性设置当前博客的标签。

博客YAML中的categories属性可以通过使用 **EXPORT_HUGO_CATEGORIES** 进行设置，或者使用标题的标签属性进行设置，不过该标签需要添加@作为前缀。


## 实时预览 {#实时预览}

如果编辑博客时，每一次更改想要在浏览器中看到修改效果，都需要先转换org文件到md文件，则会非常麻烦。因此需要实时预览，每次保存org文件后，hugo
启动的服务端会自动更新改次变更。

总共有如下两步:

1.  启动 **org-hugo-auto-export-mode** 的次要模式。启动该次要模式可以仅作用于org文件或者整个项目，仅作用于一个org文件需要在org文件最末尾添加如下代码。
    ```org
    ​* Footnotes
    ​* COMMENT Local Variables  :ARCHIVE:
    # Local Variables:
    # eval: (org-hugo-auto-export-mode)
    # End:
    ```
    上述代码添加后，需要重启该buffer,这将会确保该次要模式的启动更新有效。
2.  hugo启动服务器时，开启实时更新模式。使用如下代码启动hugo服务器:
    ```shell
    hugo server -D --navigateToChanged
    ```
    **-D** 参数将会加载草稿博客, **--navigateToChanged** 会启动实时预览。
