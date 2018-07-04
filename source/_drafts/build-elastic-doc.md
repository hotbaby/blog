---
title: 编译Elasticsearch文档
comments: true
toc: true
date: 2018-07-04 17:58:19
tags: [elasticsearch]
---

## 环境准备

下载elastic doc编译工具

```shell
git clone https://github.com/elastic/docs
```

下载elasticsearch相应版本的release

```shell
wget https://codeload.github.com/elastic/elasticsearch/tar.gz/v5.5.3
```

## 构建

```shell
cd docs; 	# 进入doc下载目录
./build_docs.pl --pdf --toc --doc ~/elasticsearch-5.5.3/docs/reference/index.asciidoc
```

> ~/elasticsearch-5.5.3为elasticsearch解压目录