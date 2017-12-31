---
title: markdown语法
date: 2017-09-27 13:16:02
tags:
---

## 标题
```markdown
  # 这是<h1> 一级标题
  ## 这是<h2> 二级标题
  ### 这是<h3> 三级标题
  #### 这是<h4> 四级标题
  ##### 这是<h5> 五级标题
  ###### 这是<h6> 六级标题
```
<!-- more -->
# 这是一级标题
## 这是二级标题
### 这是三级标题
#### 这是四级标题
##### 这是五级标题
###### 这是六级标题

### 给标题添加*class*和*id*
```markdown
# 这个标题拥有一个id {#my_id}
# 这个标题有两个classes {.class1 .class2}
```

## 强调
*这会是 斜体 的文字*
_这会是 斜体 的文字_

**这会是 粗体 的文字**
__这会是 粗体 的文字__

_你也 **组合** 这些符号_

~~这个文字将会被横线删除~~

## 无序列表
* Item 1
* Item 2
  * Item 2a
  * Item 2b
## 有序列表
1. Item 1
1. Item 2
1. Item 3
   1. Item 3a
   1. Item 3b

## 添加图片
![GitHub Logo](/images/avatar.jpg)
Format: ![Alt Text](/images/wechatpay.jpg)

## 链接
http://github.com - 自动生成！
[GitHub](http://github.com)

## 引用
正如 Kanye West 所说：

> We're living the future so
> the present is our past.

## 分割线
如下，三个或者更多的

---

连字符

***

星号

___

下划线



## 行内代码
我觉得你应该在这里使用
`<addr>` 才对。

## 代码行数
如果你想要你的代码块显示代码行数，只要添加 line-numbers class 就可以了。
```javascript {.line-numbers}
function add(x, y) {
  return x + y
}
```



## 任务列表
- [x] @mentions, #refs, [links](), **formatting**, and <del>tags</del> supported
- [x] list syntax required (any unordered or ordered list supported)
- [x] this is a complete item
- [ ] this is an incomplete item



## 表格
First Header | Second Header
------------ | -------------
Content from cell 1 | Content from cell 2
Content in the first column | Content in the second column

## 上标
30^th^

## 下标
H~2~O

## 脚注
Content [^1]

[^1]: Hi! This is a footnote

标记
==marked==

## 流程图

```flow
st=>start: 开始
e=>end: 结束
op=>operation: 我的操作
cond=>condition: 确认？

st->op->cond
cond(yes)->e
cond(no)->op
```
