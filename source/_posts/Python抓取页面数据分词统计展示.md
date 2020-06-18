---
layout: next
title: Python抓取页面数据分词统计展示
date: 2020-06-18 16:53:29
tags:
---

# Python抓取京东评论数据生成词云

> 前提:想抓取商品评论分析关键词出现频率

![aaa](https://note.youdao.com/yws/api/personal/file/94251ED148A448729CFCFE6434C6EADF?method=download&shareKey=03ff9d0b39c0895dd71e91f6ecfe6c22)


## 实现方案
- 打开京东SKU详情页面，查看评论，下一页（XHR 请求）找到评论数据接口
- 使用到的Python 组件：
    - requests,
    - jieba,
    - numpy,
    - pandas,
    - matplotlib,
    - PIL(Pillow)

## 细节步骤
- 评论接口地址拼接
- 返回数据为JSONP，字符串截取一下（可以自定义callback参数）
- 切分词
- 停用词
- 词组统计：wordData......agg(total='count')
- 数据保存：可以使用其他逻辑存档入数据库
- 词云图片生成、展示

## 小问题
- 请求数据接口频率额需要控制 不要急不要慌
- 文本读取的格式：GBK 
- `wordData......agg(`__total='count'__`)`
- 词云图生成设置的字体：挑个系统自带的 | 或者Copy你的字体文件到系统字体目录下
    -  Win：simsun.ttc  
    -  Mac:`[ ~ | /System]/Libray/Fonts`

## 完整代码
```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

import requests
import json
import re
import sys
import os
import jieba
import pandas as pd
import numpy
from wordcloud import WordCloud
import matplotlib.pyplot as plt
from os import path
import numpy as np
from PIL import Image
# 数据爬取模块


def get_comments():
    all_comments = ""
    fetchJSON_comment = "fetchJSON_comment9"
    skuID =  "1109759" # "4093841" # "100004549676"  #
    for i in range(1, 5):
        url2 = str(i)
        url1c = 'https://club.jd.com/comment/productPageComments.action?callback=' + \
            fetchJSON_comment+url2+'&productId='+skuID+'&score=0&sortType=5&page='
        url3c = '&pageSize=10&isShadowSku=0&rid=0&fold=1'

        finalurlc = url1c+url2+url3c
        xba = requests.get(finalurlc)
        # fetchJSON_comment(
        print(finalurlc, xba.text[0:len(fetchJSON_comment+url2)+1])
        data = json.loads(xba.text[len(fetchJSON_comment+url2)+1:-2])
        for j in data['comments']:
            content = j['content']
            all_comments = all_comments+content
        print(i, xba.text[0:20])
    return all_comments

# 数据清洗处理模块

xt=""

def data_clear(xt):
    xt = get_comments()
    sys.exit(xt)
    pattern = re.compile(r'[\u4e00-\u9fa5]+')
    filedata = re.findall(pattern, xt)
    xx = ''.join(filedata)
    clear = jieba.lcut(xx)   # 切分词
    cleared = pd.DataFrame({'keywords': clear})
    stopwords = pd.read_csv("chineseStopWords.txt", index_col=False,
                            quoting=3, sep="\t", names=['stopword'], encoding='GBK')
    cleared = cleared[~cleared.keywords.isin(stopwords.stopword)]
    # count_words = cleared.groupby(by=['clear'])['clear'].agg({"num": numpy.size})
    count_words = cleared.groupby('keywords')['keywords'].agg(total='count')
    count_words = count_words.reset_index().sort_values(
        by=["total"], ascending=False)
    # df = pd.DataFrame(count_words)
    # if os.path.exists("count_words.csv"):
    #     os.remove('count_words.csv')
    # df.to_csv('count_words.csv', encoding='GBK')
    xt = count_words
    return count_words

# 词云展示模块


def make_wordclound():
    # d = path.dirname(__file__)
    # msk = np.array(Image.open(path.join(d, "151.jpg")))
    word_frequence = {x[0]: x[1] for x in data_clear(xt).head(200).values}
    wordcloud = WordCloud(font_path="simsun.ttc", # mask=msk,
                          background_color="#EEEEEE", max_font_size=250, width=2100, height=1200)
    wordcloud = wordcloud.fit_words(word_frequence)
    plt.imshow(wordcloud)
    plt.axis("off")
    plt.show()


if __name__ == "__main__":
    make_wordclound()

```
