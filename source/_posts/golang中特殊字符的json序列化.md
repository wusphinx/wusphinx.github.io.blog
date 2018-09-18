---
title: golang中特殊字符的json序列化
date: 2018-06-19 01:40:14
tags:
---


先来看一段`golang`
```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {

	data := map[string]string{
		"str0": "Hello, world",
		"str1": "<",
		"str2": ">",
		"str3": "&",
	}
	jsonStr, _ := json.Marshal(data)

	fmt.Println(string(jsonStr))
}
```
输出结果
```
{"str0":"Hello, world","str1":"\u003c","str2":"\u003e","str3":"\u0026"}
```

先来段`rust`的
```rust
extern crate rustc_serialize;
use rustc_serialize::json;
use std::collections::HashMap;

fn main(){
    let mut data =  HashMap::new();
    data.insert("str0","Hello, world");
    data.insert("str1","<");
    data.insert("str2",">");
    data.insert("str3","&");
    println!("{}", json::encode(&data).unwrap());
}
}
```
结果
```
{"str0":"Hello, world","str2":">","str1":"<","str3":"&"}
```
再来看段`python`的
```python
import json

data = dict(str0='Hello, world',str1='<',str2='>',str3='&')

print(json.dumps(data))
```
输出结果
```
{"str0": "Hello, world", "str1": "<", "str2": ">", "str3": "&"}
```

再看看java的
```java
import org.json.simple.JSONObject;

class JsonDemo
{
    public static void main(String[] args)
    {
        JSONObject obj = new JSONObject();

        obj.put("str0", "Hello, world");
        obj.put("str1", "<");
        obj.put("str2", ">");
        obj.put("str3", "&");

        System.out.println(obj);
    }
}
```
输出结果
```
{"str3":"&","str1":"<","str2":">","str0":"Hello, world"}
```

可以看到`python`、`rust`和`java`对这4个字符串序列化结果几乎是相同的了(除了java序列化后顺序有微小变化外），golang明显对 *&lt;* ,
*&gt;* , *&* 进行了转义处理，看看文档怎么说的

>// String values encode as JSON strings coerced to valid UTF-8,  
// replacing invalid bytes with the Unicode replacement rune.  
// The angle brackets "<" and ">" are escaped to "\u003c" and "\u003e"  
// to keep some browsers from misinterpreting JSON output as HTML.  
// Ampersand "&" is also escaped to "\u0026" for the same reason.  

*&* 被转义是为了防止一些浏览器将JSON输出曲解为HTML，
而 *&lt;* ,*&gt;* 被强制转义是因为golang认为这俩是无效字节（这点比较奇怪），
我如果技术栈都是golang还好说，如果跨语言跨部门合作一定需要注意这点（已踩坑）……
