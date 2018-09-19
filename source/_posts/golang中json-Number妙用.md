---
title: golang中json.Number妙用
date: 2018-07-17 21:03:14
tags:
---

最近跟某斯调试一个API接口，接口返回数据是`json`格式 ，按文档描述是一个整型数据，于是定义如下
```go
	type Data struct {
		Api int `json:"api"`
	}
```
在***入参相同***的情况下，第一次调用，得到的结果是：
```go
{"api":1}
```
然而第二次调用，得到结果却是：
```go
{"api":"1"}
```
与对方开发人员沟通后发现这是一个bug，由于流程问题，没办法立即修改上线，想想还是我做兼容比较好，效果是既能解析于`{"api":1}`，也能够解析`{"api":"1"}`，于是我想到了`json.Number`，[Number类型定义](https://tools.ietf.org/html/rfc7159#section-6)如下
>The representation of numbers is similar to that used in most
   programming languages.  A number is represented in base 10 using
   decimal digits.  It contains an integer component that may be
   prefixed with an optional minus sign, which may be followed by a
   fraction part and/or an exponent part.  Leading zeros are not
   allowed.

简单翻译一下：`Number`可以表示十进制数，但是前导零数据是不允许的（好吧，此处欠妥，还是看示例）比如这种`012`、`-012`等等
我们重新调整数据结构
```go
	type Data struct {
		Api json.Number `json:"api"`
	}
```
数据解析
```go
	var d Data
	err := json.Unmarshal([]byte(`{"api":"12"}`), &d)
```
实际上我们遇到的问题是数据肯定是十进制数，但不确定有没有`""`号，而`json.Number`帮助抽象了数据这层概念，在确保类型的前提下，由调用方自己决定最终使用的类型
```go
// String returns the literal text of the number.
func (n Number) String() string { return string(n) }

// Float64 returns the number as a float64.
func (n Number) Float64() (float64, error) {
	return strconv.ParseFloat(string(n), 64)
}

// Int64 returns the number as an int64.
func (n Number) Int64() (int64, error) {
	return strconv.ParseInt(string(n), 10, 64)
}
```
我可以安全的使用
```
	d.Api.Int64()
```
从而得到自己想要的类型。
当然，我的问题还可以用`RawMessage`类型解决
```
	type Data struct {
		Api json.RawMessage `json:"api"`
	}
```
文档释义如下：
```
// RawMessage is a raw encoded JSON value.
// It implements Marshaler and Unmarshaler and can
// be used to delay JSON decoding or precompute a JSON encoding.
type RawMessage []byte
```
确定字段名，拿到`json`里面的原始数据值，再转化成自己想要的类型。
常读源码，常读常新