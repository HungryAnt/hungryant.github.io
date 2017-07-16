---
layout: post
title:  "white-space: pre-line 实现文本换行"
date:   2017-07-16 13:05:42 +0800
categories: css
---

学习前端开发时看到一个渐变设置文本换行的例子，记录下来

如下是VUE教程中的实例

[表单控件绑定](https://cn.vuejs.org/v2/guide/forms.html)

```html
<span>Multiline message is:</span>
<p style="white-space: pre-line">{{ message }}</p>
<br>
<textarea v-model="message" placeholder="add multiple lines"></textarea>
```

在p中添加样式 `white-space: pre-line` 即可保留原始文本中的换行符

[CSS white-space 属性 W3school](http://www.w3school.com.cn/cssref/pr_text_white-space.asp)
