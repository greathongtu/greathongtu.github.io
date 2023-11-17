+++
title = 'Vim 进阶'
date = 2023-11-16T14:39:47+08:00
draft = false
tags = ['vim', 'editor']
+++

## Repeat using . and ;

```javascript
// use ' to revert ;, u to revert .
var foo = "method("+argument1+","+argument2+")";
```

* f+ to **Move**, s\<space\>+\<space\>\<Esc\> to **Change**.
* use ; to repeat search and . to repeat change

```javascript
var foo = "method(" + argument1 + "," + argument2 + ")";
```
<br>

### Dot Formula
The Ideal:
* one keystroke to **Move**, one keystroke to **Execute**. 
* Act, Repeat, Reverse
* Operator + Motion = Action (dw、guaw、gcap)


## Misc
* w: word, s: sentence, p: paragraph, 一般来说，d{motion} 命令和 aw、as 和 ap 配合使用比较好，而 c{motion} 和 iw is ip一起用效果会更好
* :!ls exec shell command inside vim, % means current file
* use * to **search**, cw\<something\>\<Esc\> toChange**.
* 1. modify column data: visual select then c, finally \<Esc\>
    2. 非方形区域也行，如在行尾添加分号：选中block,go to end using $
* :w !sudo tee % > /dev/null 以超级用户权限保存文件
* :%s/Practical/Pragmatic/ 每行内的第一个“Practical”替换为“Pragmatic”
* gu{motion} to make all lowercase
* in visual mode: o: Goto other end of highlighted text.
* in visual mode: r-: change them all to -----
* in insert mode: \<C-o\>zz to get into insert normal mode.(move curr line to center and continue insert)
* 18\<C-x\>: find the first number later in this line, -18 to it.
* in insert mode: \<C-w\> delete back one word. \<C-u\> delete back to start of line
* in insert mode: \<C-r\>=6*24\<CR\> insert the result
* R to replace mode
