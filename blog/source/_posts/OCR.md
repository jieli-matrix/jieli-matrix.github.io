---
title: paddleocr多平台使用
date: 2021-02-14 18:43:03
tags:
    - paddle
    - ocr
categories:
    - 技术
---

早在上学期就听说百度paddle开源了ocr模型，于是今天在帮爸爸打印文档时尝试了paddleocr，在这篇文章里记录下使用时遇到的问题。

<!--more-->

由于时间所限，所以仅通过pypi的paddleocr进行OCR识别。

## windows平台

根据官网的安装步骤，仅需要pip install "paddleocr>=2.0.1"即可。但事实上，当这样安装后，在python终端内
``` python
from paddleocr import PaddleOCR
```
会出现报错"No module named 'paddle'"
正确的安装思路如下

``` shell
python3 -m pip install paddlepaddle-gpu==2.0.0 -i https://mirror.baidu.com/pypi/simple # gpu机器
python3 -m pip install paddlepaddle==2.0.0 -i https://mirror.baidu.com/pypi/simple # cpu机器
pip install "paddleocr>=2.0.1"
```

再次进入python终端内

``` python
from paddleocr import PaddleOCR
```
会出现报错“conda环境下缺少geos_c.dll”，这说明缺少geos_c动态链接库，从该[网址](https://www.lfd.uci.edu/~gohlke/pythonlibs/#shapely)上下载Shapely‑1.7.1‑cp39‑cp39‑win_amd64.whl，并更改后缀为rar，解压Shapely‑1.7.1‑cp39‑cp39‑win_amd64.rar文件，找到geos_c.dll文件，并复制到对应的conda环境Libary下。（以我的配置为例，为D:\opt\Miniconda3\envs\ocr\Library\bin）

再次import PaddleOCR则正常，但运行时仍会报错“Initializing libiomp5md.dll, but found libiomp5md.dll already initialized”,这时候在python文件头写入如下设置

``` python
import os os.environ['KMP_DUPLICATE_LIB_OK']='True'
```
就可以成功运行paddleocr啦~

这里贴一个官网的调用示例：

``` python
from paddleocr import PaddleOCR,draw_ocr
# Paddleocr supports Chinese, English, French, German, Korean and Japanese.
# You can set the parameter `lang` as `ch`, `en`, `french`, `german`, `korean`, `japan`
# to switch the language model in order.
ocr = PaddleOCR(use_angle_cls=True, lang='en') # need to run only once to download and load model into memory
img_path = 'PaddleOCR/doc/imgs_en/img_12.jpg'
result = ocr.ocr(img_path, cls=True)
for line in result:
    print(line)
```
返回是list形式，按文本行（横排或竖排）返回，包含文本框位置，置信度和文字内容。


## Linux平台

Linux平台下安装很简单，分别安装paddlepaddle和paddleocr即可。

``` shell
python3 -m pip install paddlepaddle-gpu==2.0.0 -i https://mirror.baidu.com/pypi/simple # gpu机器
python3 -m pip install paddlepaddle==2.0.0 -i https://mirror.baidu.com/pypi/simple # cpu机器
pip install "paddleocr>=2.0.1"
```

补充：paddleocr的[pypi安装文档](https://github.com/PaddlePaddle/PaddleOCR/blob/release/2.0/doc/doc_ch/whl.md)内容不够完全，windows或mac平台直接安装可能会出现较多报错，建议先看[快速安装](https://github.com/PaddlePaddle/PaddleOCR/blob/release/2.0/doc/doc_ch/installation.md)文档。

参考链接:
[缺少geos_c.dll文件](https://www.cnblogs.com/xuanmanstein/p/13840670.html)
[OpenMP runtime have been linked into the program](https://blog.csdn.net/qq_45266796/article/details/109028605)