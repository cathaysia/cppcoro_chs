# 介绍

这是一个 **非官方** 发布的翻译，其原仓库位于 https://github.com/lewissbaker/cppcoro 。主要目的是让大家读得更加舒服。

# 构建

与原仓库不同的是，我采用了 Sphinx + reStructedText 的方式撰写文档，主要原因是不想手动写目录。。。
   
文档为 index.rst

Sphinx 是 reStructedText 的超集。这意味着 Github 上虽然也可以看到 rst 的预览，但是少量标记 Github 是无法识别的。鄙人还是建议 clone 到本地自己构建一下。

环境需要 Python3

进入 clone 路径，执行 ：

1. ``pip install -r requirements.txt``
2. ``make html``

如果需要构建 PDF 、EPUB 的话，可以分别使用 ``make pdf`` 和 ``make epub`` 。生成的文件位于 ``_build`` 目录下。
