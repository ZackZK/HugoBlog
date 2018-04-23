---
date: 2016-08-26T00:00:00Z
description: null
modified: 2016-08-26
tags:
- python
- datetime
title: python 时间类型操作
---

经常碰到需要对日期时间的字符串操作，查了又经常忘记，所以写到这里当作自己的笔记。

datetime模块里有几个经常用到类, datetime, date, time

+ date 日期类 (年，月，日)
+ time 时间类 (小时， 分钟， 秒， 微秒)
+ datetime 日期 + 时间

## datetime类 ##

### 从字符串到datetime ###
* strptime函数
{{< highlight python >}}

>>> datetime.datetime.strptime('06-05-2010', "%d-%m-%Y").date()
datetime.date(2010, 05, 06)

{{< / highlight >}}

* 第三方库 dateutil
{{< highlight python >}}

from dateutil import parser
dt = parser.parse("Aug 28 1999 12:00AM")

{{< / highlight >}}

可以通过pip命令安装dateutil安装
{{< highlight python >}}
pip install python-dateutil
{{< / highlight >}}


-------------------------------------------------------------------------------

| 格式化 | 意义                                                             | 例子                        |
|--------+------------------------------------------------------------------+-----------------------------|
| %d     | Day of the month as a zero-padded decimal number.前面填充0的日期 | 比如00, 01, ..., 31         |
| %m     | Month as a zero-padded decimal number. 月份                      | 00,01,...12                 |
| %y     | 去掉century的两位年数                                            | 00, 01, ..., 99             |
| %Y     | 4位数年                                                          | 1991, 2016                  |
| %H     | 小时(24) Hour (24-hour clock) as a zero-padded decimal number.   | 00, 01, ..., 23             |
| %I     | 小时(12) Hour (12-hour clock) as a zero-padded decimal number. | 01, 02, ..., 12             |
| %M     | 分钟 Minute as a zero-padded decimal number.                   | 00, 01, ..., 59             |
| %S     | 秒 Second as a zero-padded decimal number.                      | 00, 01, ..., 59             |
| %f     | 毫秒 Microsecond as a decimal number, zero-padded on the left.   | 000000, 000001, ..., 999999 |
|        |                                                                  |                             |

-------------------------------------------------------------------------------


### datetime到字符串 ###
* strftime 函数
{{< highlight python >}}

>>> a = datetime.datetime(2010, 12, 30)
>>> a.strftime("%Y-%m-%d")
'2010-12-30'

{{< / highlight >}}

### 获取当前时间 ###
{{< highlight python >}}

>>> datetime.datetime.now()
datetime.datetime(2016, 8, 29, 14, 55, 40, 684000)

{{< / highlight >}}

# time模块 #
