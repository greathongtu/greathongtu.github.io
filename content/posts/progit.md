+++
title = 'Progit'
date = 2024-04-27T19:01:54+08:00
draft = false
+++

#### git log:
	* --grep: 仅显示commit msg中包含指定字符串的提交。
	* -S string: 仅显示添加或删除匹配string的提交
	* -- some_dir 最后可以加上搜索的目录名，中间用-- 分隔

#### 撤回修改
* commit 之后想修改： amend更改刚才的commit message
```console
$ git commit -m 'initial commit'
$ git add forgotten_file
$ git commit --amend
```
* add之后想撤回：
```console
git reset HEAD file_name
```
* 删掉修改：
```console
git checkout -- file_name
```
* fetch 和 merge 的区别： fetch 拉到本地repo但是不merge
* git log --oneline --decorate --graph --all 看分支情况
