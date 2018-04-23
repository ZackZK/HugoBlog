---
date: 2016-01-21T00:00:00Z
description: 网络爬虫
modified: 2016-01-21
tags:
- python
- requests
- 爬虫
title: 使用requests的网络爬虫
---

requests 帮助主页： http://docs.python-requests.org/en/latest/user/quickstart/

# 1. requests的json支持
requests本身自带了json支持，不需要额外的json库来做转换。比如得到的内容是json格式，可以使用

``` python
import requests

r = requests.get('https://api.github.com/events')

r.json()

[{u'repository': {u'open_issues': 0, u'url': 'https://github.com/...

```

# 2. form-encoded data

requests库可以简单的将dict类型传递给get或post方法的data参数就可以实现自动实现传递form-encoded参数。但是，如果参数本身是一个数组的话，比如网站需要传下面的POST参数：  
![post_array](/images/python_requests/post_form_encoded_array.png)

可以看出来，amarket和coupon_descr都是数组类型，有多个数值，这个时候需要通过将list类型赋值给对应的dict key，代码如下：

```python
data = {"is_funda_search":"0",

        "fundavolume":"100",

        "amarket[]":["sh", "sz"],

        "coupon_descr[]": ["+3.0%", "+3.2%", "+3.5%", "+4.0%", "other"],

        "rp":"50",

        "maturity":""}

content = s.post(url, data=data)
```