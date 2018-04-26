---
date: 2016-11-18T00:00:00Z
description: emacs使用笔记
modified: 2016-11-18
tags:
- emacs
- elisp
- 代理
title: emacs 笔记
---

# emacs 设置代理 #
```lisp
    (setq url-proxy-services
       '(("no_proxy" . "^\\(localhost\\|10.*\\)")
         ("http" . "proxy.com:8080")
         ("https" . "proxy.com:8080")))
```

# emacs 书签 #
与存储光标位置的寄存器略有不同
书签可以使用单词来命名，而不限于一个字符。起一个容易记住的名字
退出 Emacs 后，书签不会消失，下次还可以使用

```lis
C-x r m (name)	M-x bookmark-set	设置书签

C-x r b (name)	M-x bookmark-jump	跳转到书签

C-x r l	M-x bookmark-bmenu-list	书签列表

M-x bookmark-delete	删除书签

M-x bookmark-load	读取存储书签文件
```

书签默认存储在 ~/.emacs.bmk 文件中
在配置文件中，可以设置书签存储的文件

``` lisp
;; 书签文件的路径及文件名

(setq bookmark-default-file "~/.emacs.d/.emacs.bmk")

;; 同步更新书签文件 ;; 或者退出时保存

(setq bookmark-save-flag 1)
```



