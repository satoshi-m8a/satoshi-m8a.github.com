---
layout: post
title: "Scalaで無名関数を定義して即実行"
date: 2013-05-17 22:37
comments: true
categories: 
---

JavaScriptでこんなことしますよね。
``` javascript
var b = 1;
(function(a){
	console.log(a + b);
	b++;
})(10);
```

これと同じようなことをScalaでしたければ
以下のようにすればできます。
``` scala
var b = 1
((a:Int) => {
	println(a + b)
	b += 1
})(10)
```