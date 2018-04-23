---
date: 2015-07-23T00:00:00Z
description: 介绍python的package中如何操作相对路径文件
modified: 2015-07-13
tags:
- python
- relative path
title: python 相对路径文件的操作
---

python项目中，如果pyton代码需要访问某个外部文件，该文件位于代码文件的某个相对路径位置，我们可以在代码中使用相对路径来访问该文件。
比如图中的代码结构：
![files]({{ site.url }}/images/python_relative_file_path/files_structure.PNG)

sample.py文件中，如果要访问配置文件server.ini文件，就可以用 "../conf/server.ini"来进行访问。

但是经常的问题是，该python文件又被别的目录的python文件import引用， 此时相对路径就会出错。这是因为此时相对路径是基于当前运行脚本的路径来计算的，
如果被引用的python文件和调用文件不在同一个目录，则相对路径就会失效。

比如，我们的例子中， sample.py文件内容如下:

{{< highlight python >}}
import ConfigParser
import os

def read_conf():
    file_path = '../conf/server.ini'
    config = ConfigParser.ConfigParser()
    config.read(file_path)
    return config.get('conf', 'value')


if __name__ == '__main__':
    print read_conf()
{{< / highlight >}}


main.py内容如下:

{{< highlight python >}}
from utils.sample import read_conf

print read_conf()
{{< / highlight >}}

比如上图中，如果main.py调用read_conf时就会发现server.ini文件找不到。问题就处在运行main.py时，当前路径是main.py所在的文件夹，而sample.py中使用的相对
路径基于该文件夹就会找错位置。解决办法是在sample.py中使用文件绝对路径来访问server.ini文件， 但是我们又要根据脚本存放的当前位置来获得运行时的路径。这个时候
我们就需要用到python中的 **\__file__**变量， 该内置变量存放了本python文件的当前路径信息，根据该变量，我们可以**os.path.dirname(\__file__)**得到相应文件
的绝对路径。

在sample.py中file_path变量做如下修改：

{{< highlight python >}}
    file_path = '../conf/server.ini'
    -->
    file_path = os.path.join(os.path.dirname(__file__) + '/../conf/server.ini')
{{< / highlight >}}

*注意* 路径前加 '/',否则文件路径会不完整。

参考资料：

<http://stackoverflow.com/questions/918154/relative-paths-in-python>
<http://stackoverflow.com/questions/1270951/python-how-to-refer-to-relative-paths-of-resources-when-working-with-code-repo>

    
