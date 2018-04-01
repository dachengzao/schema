## schema
基于gorilla修改，不可用于生产环境。

Package gorilla/schema converts structs to and from form values.

## install

	go get github.com/dachengzao/schema
	
## 解读

不要被 schema 这个名字迷惑, 这其实是一个 form 提交数据到 struct 实例的转换器. 非常实用的一个功能. 使用也非常简单. 但是如果您查看源码发现没有引入 net/http 包, 这就是解耦. 因为 http.Request.Form 其实就是一个 map[string][]string 类型.

```go
package main

import (
    "fmt"
    "github.com/gorilla/schema"
)

type Person struct {
    Name  string
    Phone string
}

func main() {
    // 模拟一个 Form 数据
    values := map[string][]string{
        "Name":  {"John"},
        "Phone": {"999-999-999"},
    }
    person := new(Person)
    decoder := schema.NewDecoder()
    decoder.Decode(person, values) // 从Form数据 SetTo *Person
    fmt.Printf("%#v\n", person)
}
```

是不是很简单, 其代码实现也很简单. 而且 Schema 还提供了注册转换函数

```go
type Converter func(string) reflect.Value
func (d *Decoder) RegisterConverter(value interface{}, converterFunc Converter)
```

比如我们常见的 time.Time 类型, 因为 layout 的多样性, 可以根据我们的应用写独立的转换函数

```go
func StringToTime(s string) reflect.Value {
    t, _ := time.Parse("2006-01-02", s)
    return reflect.ValueOf(t)
}
type Person struct {
    Name  string
    Phone string
}

func main() {
    // 模拟一个 Form 数据
    values := map[string][]string{
        "Name":  {"John"},
        "Phone": {"999-999-999"},
            "Day":     {"2013-02-01"},
    }
    person := new(Person)
    decoder := schema.NewDecoder()
    decoder.RegisterConverter(time.Now(), StringToTime)
    decoder.Decode(person, values) // 从Form数据 SetTo *Person
    fmt.Printf("%#v\n", person)
}
```

实用, 简单, 解耦

## Example

Here's a quick example: we parse POST form values and then decode them into a struct:

```go
// Set a Decoder instance as a package global, because it caches 
// meta-data about structs, and an instance can be shared safely.
var decoder = schema.NewDecoder()

type Person struct {
    Name  string
    Phone string
}

func MyHandler(w http.ResponseWriter, r *http.Request) {
    err := r.ParseForm()
    if err != nil {
        // Handle error
    }

    var person Person
    
    // r.PostForm is a map of our POST form values
    err := decoder.Decode(&person, r.PostForm)
    if err != nil {
        // Handle error
    }

    // Do something with person.Name or person.Phone
}
```

Conversely, contents of a struct can be encoded into form values. Here's a variant of the previous example using the Encoder:

```go
var encoder = schema.NewEncoder()

func MyHttpRequest() {
    person := Person{"Jane Doe", "555-5555"}
    form := url.Values{}

    err := encoder.Encode(person, form)

    if err != nil {
        // Handle error
    }

    // Use form values, for example, with an http client
    client := new(http.Client)
    res, err := client.PostForm("http://my-api.test", form)
}

```

To define custom names for fields, use a struct tag "schema". To not populate certain fields, use a dash for the name and it will be ignored:

```go
type Person struct {
    Name  string `schema:"name"`  // custom name
    Phone string `schema:"phone"` // custom name
    Admin bool   `schema:"-"`     // this field is never set
}
```

The supported field types in the struct are:

* bool
* float variants (float32, float64)
* int variants (int, int8, int16, int32, int64)
* string
* uint variants (uint, uint8, uint16, uint32, uint64)
* struct
* a pointer to one of the above types
* a slice or a pointer to a slice of one of the above types

Unsupported types are simply ignored, however custom types can be registered to be converted.

More examples are available on the Gorilla website: http://www.gorillatoolkit.org/pkg/schema

## License 

BSD licensed. See the LICENSE file for details.
