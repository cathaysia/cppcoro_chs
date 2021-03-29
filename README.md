# 介绍

[![文档的构建情况](https://github.com/cathaysia/cppcoro_chs/actions/workflows/build_docs.yml/badge.svg)](https://github.com/cathaysia/cppcoro_chs/actions/workflows/build_docs.yml)
[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/cathaysia/cppcoro_chs/blob/master/LICENSE)

这是一个 **非官方** 发布的翻译，其原仓库位于 https://github.com/lewissbaker/cppcoro 。主要目的是让大家读得更加舒服。

文档的 page 页面：  https://cathaysia.github.io/cppcoro_chs/

# 构建

与原仓库不同的是，我采用了 Sphinx + reStructedText 的方式撰写文档，主要原因是不想手动写目录。。。
   
文档为 `index.rst`

Sphinx 是 reStructedText 的超集。这意味着 Github 上虽然也可以看到 rst 的预览，但是少量标记 Github 是无法识别的。所以我推荐使用 Github Pages 方式查看或者直接 clone 到本地构建一下。

环境需要 Python3

进入 clone 路径，执行 ：

1. ``pip install -r requirements.txt``
2. ``make html``

如果需要构建 PDF 、EPUB 的话，可以分别使用 ``sphinx-build -b latexpdf . _build/latex`` 和 ``sphinx-build -b epub . _build/epub`` 。生成的文件位于 ``_build`` 目录下。
