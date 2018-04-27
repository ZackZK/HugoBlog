---
date: 2018-4-27T00:00:00Z
description: go语言的坑
modified: 2018-04-27
tags:
- golang
title: golang的坑
---

# 1\. json.Marshal函数#

* 返回空" {}"

  比如

  ```go
  type TestObject struct {
      kind string `json:"kind"`
      id   string `json:"id, omitempty"`
      name  string `json:"name"`
      email string `json:"email"`
  }

  testObject := TestObject{
      "TestObj",
      "id",
      "Your name",
      "email@email.com"
  }
  fmt.Println(testObject)
  b, err := json.Marshal(testObject)
  fmt.Println(string(b[:]))
  ```

  **结果** ：

  ```Go
  {TestObject id Your name email@email.com}
  {}
  ```

  **原因**：

  golang中使用字母是否大写定义导出， encoding/json库会忽略非导出的字段。

  正确方法：

  导出字段使用大写字母开头，如:

  ```go
  type TestObject struct {
      Kind string `json:"kind"`
      Id   string `json:"id, omitempty"`
      Name  string `json:"name"`
      Email string `json:"email"`
  }
  ```

  ​

* []byte 类型字段的Marshal

  ```go
  func main(){
     type foo struct {
        Data []byte `json:"data"`
     }

     bar := foo{[]byte{1}}
     body, _ := json.Marshal(bar)

     fmt.Printf(string(body))
  }
  ```

  运行结果竟然是：

  ```shell
  {"data":"AQ=="}
  ```

  golang Marshal文档上关于[]byte类型的说明：

  > Array and slice values encode as **JSON** arrays, except that []**byte** encodes as a base64-encoded string.

   也就是说*[]byte* 类型作为base64的字符串进行encoding, 所以才出现上面奇怪的结果。