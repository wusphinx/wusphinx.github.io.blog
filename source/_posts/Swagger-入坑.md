---
title: "Swagger 入坑"
date: 2017-06-02T17:49:36+08:00
---

记得早初入公司的时候， 和前端的同事对接口，因为坐的近的缘故，直接口头定一下参数，
然后用QQ交流下，调试，交付，因为沟通很便利，倒也没有遇到太大的障碍，后来公司逐渐
发展，人员有流动，队伍在壮大，头口定接口的方式的弊端立马就显现出来了，于是小组里面
搭建了一个文档服务，大家都开始用markdown写接口，感觉高大上的感觉，但也是简陋，写的
比较粗糙，类似于下面这种：

```
    Url: www.example.com

    Method: POST

    Request:

        page int required
        limit int required

    Response:

        {
            status: ok
            ...
        }
```

每次客户端要接口很紧，写完就立马去实现接口了，当然，因为后续接口可能会有一些变化，也得先改文档
再更新代码，有时候忘了，也就不了了之了，总之感觉就是用着很不爽。

后来团队转到Golang，自己看了下{% link beego https://beego.me/ %},发现了{% link swagger https://swagger.io/ %}这个东西，觉得很不错的样子，而且Beego的作者将beego融入到
swagger里面去了，很是不错，觉得写代码即是文档这个概念很好，很规范，很可惜团队最终框架选型没有
用Beego。不过还好团队的里面有个牛人LD,研究了下swagger下，并最终在团队中推广开来。

官方说法：Swagger是一个规范和完整的框架，用于生成、描述、调用和可视化 RESTful 风格的 Web 服务。总体目标是使客户端和文件系统作为服务器以同样的速度来更新。文件的方法，参数和模型紧密集成到服务器端的代码，允许API来始终保持同步。

个人觉得swagger最大的好处是实现了可文档化的代码编写和注释方法，我们写代码总是要写注释，依照swagger的规范，我们能写出可文档化的注释，岂不妙哉！
swagger的golang文档地址goswagger
以下是是根据goswagger文档描述写的一个demo:

```go
// Package classification Petstore API.
//
// the purpose of this application is to provide an application
// that is using plain go code to define an API
//
// This should demonstrate all the possible comment annotations
// that are available to turn go code into a fully compliant swagger 2.0 spec
//
// Terms Of Service:
//
// there are no TOS at this moment, use at your own risk we take no responsibility
//
//     Schemes: http
//     Host: localhost
//     BasePath: /
//     Version: 0.0.1
//     License: MIT http://opensource.org/licenses/MIT
//     Contact: John Doe<john.doe@example.com> http://john.doe.com
//
//     Consumes:
//     - application/x-www-form-urlencoded
//       - application/json
//
//     Produces:
//     - application/json
//
//
// swagger:meta
package main

import (
    "github.com/gin-gonic/gin"
)

// swagger:parameters loginForm
type LoginForm struct {
    // 用户名
    // required true
    User string `form:"user" binding:"required"`

    // 密码
    // required true
    Password string `form:"password" binding:"required"`
}

// 请求响应
// swagger:response commonResponse
type CommonResponse struct {
    // in: body
    Body struct {
        Status      string
        Description string
    }
}

func main() {
    router := gin.Default()

    // swagger:route POST /login api loginForm
    //
    // Lists pets filtered by some parameters.
    //
    // This will show all available pets by default.
    // You can get the pets that are out of stock
    //
    //     Consumes:
    //     - application/json
    //
    //     Produces:
    //     - application/json
    //
    //     Schemes: http, https
    //
    //     Responses:
    //       default: commonResponse

    router.POST("/login", func(c *gin.Context) {
        var form LoginForm
        response := CommonResponse{}.Body
        // in this case proper binding will be automatically selected
        if c.Bind(&form) == nil {
            if form.User == "user" && form.Password == "password" {
                response.Status = "Ok"
                response.Description = "you are logged in"
                c.JSON(200, response)
            } else {
                response.Status = "Error"
                response.Description = "unauthorized"
                c.JSON(401, response)
            }
        }
    })
    router.Run(":18080")
}
```

将注释导出一个json文件

``` bash
$ swagger generate spec -o ./hello.json
```

为避免语法错误，应使用{% link swagger-editor https://swagger.io/swagger-editor/ %}进行预览，
我只能说so beautiful!
写规范的代码，不断review!
