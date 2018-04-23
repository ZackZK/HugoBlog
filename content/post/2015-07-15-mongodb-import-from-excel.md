---
date: 2015-07-15T00:00:00Z
description: 从excel导入数据到mongodb中
modified: 2015-07-21
tags:
- mongodb
title: mongodb从excel中导入数据
---

mongoimport工具可以从指定的CSV， TSV 或者 JSON数据中导入到mongoDB。 而excel表格可以直接另存为csv格式。

{{< highlight bash >}}
$ mongoimport
connected to: 127.0.0.1
no collection specified!
Import CSV, TSV or JSON data into MongoDB.
 
options:
  -h [ --host ] arg       mongo host to connect to ( <set name>/s1,s2 for sets)
  -u [ --username ] arg   username
  -p [ --password ] arg   password
  -d [ --db ] arg         database to use
  -c [ --collection ] arg collection to use (some commands)
  -f [ --fields ] arg     comma separated list of field names e.g. -f name,age
  --file arg              file to import from; if not specified stdin is used
  --drop                  drop collection first 
  --upsert                insert or update objects that already exist
{{< / highlight >}}

所以，我们可以先将execl文件转换成csv文件，再利用mongoimport将其导入到mongodb中。步骤如下：

1\. 将excel文件转换成csv文件格式
    首先要保证excel第一行为**表头并为英文**，这是因为第一行会作为mongodb字段名和后面各行进行对于。再另存为csv文件：
    ![step1]({{ site.url }}/images/mongodb_import_from_csv/excel_export_step1.png)
    选择MS-DOS CSV格式:
    ![step2]({{ site.url }}/images/mongodb_import_from_csv/excel_export_step2.png)
    告警忽略:
    ![step3]({{ site.url }}/images/mongodb_import_from_csv/excel_export_step3.png)
	
2\. 将csv文件转成UTF-8编码
	可以使用各种工具，这里使用的Notepad++。 使用notepad++将上一步骤中产生的csv文件打开，选择菜单栏中转换为UTF-8选项并保存。 记得先关掉上一步中的excel。
    ![step4]({{ site.url }}/images/mongodb_import_from_csv/excel_export_step4.png)
	
3\. 将转换好的csv文件上传到linux机器上，执行下面命令：
    {{< highlight bash >}}
    mongoimport -d <your DB name > -c <collection name> --type csv --headerline --file xxx.csv
    {{< / highlight >}}
    
将上面命令中换成需要导入的db 和 对应的collection。 如果导入成功会有如下输出：

{{< highlight bash >}}
connected to: 127.0.0.1
Wed Apr 10 13:26:12 imported 60 objects
{{< / highlight >}}

### update on 2015/07/21
**mongoimport 导入数据时的数据类型问题**
在使用mongoimport可以很方便的导入需要的数据，但是导入之后会发现数字型的值都会自动变为数值类型， 即使使用引号将其括起来。这对于代码出来来说，十分不方便。
[stack overflow的讨论](http://stackoverflow.com/questions/24223443/mongoimport-choosing-field-type)

一种方法，可以通过下面的python代码将csv文件读入，然后转换成自己想要的格式，将其写入mongodb。注意：需要[PyMongo](http://api.mongodb.org/python/current/)支持。

{{< highlight python >}}
import os
import csv

class MyType:
    def __init__(self, type, value):
        self.type = type
        self.value = value

mongodb_link = 'mongodb://127.0.0.1:27017'
mongoClient = MongoClient(mongodb_link)
db = mongoClient.mytype_infos

def write_mongoDB(mytype):
    db.mytype_info.insert(mytype.__dict__)

with open('xxxx.csv') as csvfile:
    reader = csv.DictReader(csvfile)
    for row in reader:
        one = MyType(type=int(row['type']),
                     value=float(row['value']))
        write_mongoDB(one)
{{< / highlight >}}

        
    
    




