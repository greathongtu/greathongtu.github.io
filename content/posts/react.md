+++
title = 'React'
date = 2024-03-14T19:42:12+08:00
draft = false
tags = ['frontend', 'react']
+++



* jsx：component返回的jsx，看起来像html，实际是javascript，通过babel将jsx编译成javascript，这些js最后通过dom操作生产html。jsx融合了html，css，js三个。
* props：是属性，是上级component向下级component传递的只读的信息
* state：使用useState()返回一个初始值v和一个function setV。useState是一个react Hook。根据当前state更新新的state需要用lambda function
* UI = 很多components = f(state)
* Npx create-react-app@5 travel-list 运行：npm install; npm start
