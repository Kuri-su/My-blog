# ProtoBuf

go主要实现 golang/protobuf

go protobuf增强库 gogo/protobuf

## 序列化

* **ProtoBuf**

* `Json`
* `Yaml`
* `Toml`
* `PropertyList(Apple)`
* `Bson`
* `MessagePack`
* ...

## Proto3

### what's this

proto 文件是一个服务对外的 接口声明, 以及请求文档,

### 主要改变

* 移除了 `required` 字段
* 移除了缺省值
* 添加了 map 类型

### 基本例子

```protobuf
syntax = "proto3";

message SearchRequest {
	string query = 1;
	int32 page_number = 2;
	int32 result_per_page = 3; 
}
```

字段是以如下格式定义的

```protobuf
[ "repeated" ] type fieldName "=" fieldNumber [ "[" fieldOptions "]" ] ";" 
// repeated     string  query  =   1 ;
```

而这个例子是一个简单的例子, 采用了 如下格式的定义

```protobuf
type fieldName "=" fieldNumber
```

这是一个较为简单的结构, 如果对于复杂的结构, 前面可以定义为 `repeated` , 意为 重复, 实为对应元素的数组, 而在序号(`fieldNumber`) 后可以定义一些可选项.

而对于复杂的字段定义, 可以有一些复杂的字段定义, 例如下面这些, 在后面会详细讲解

* oneOf
* map
* reserved
* enum

// TODO 这里需要加上解释

你可以使用 如下命令将上面的 的 `proto` 文件编译成 Go 代码

```shell
$ protoc -I=. -I=/usr/local/include -I=$(GOPATH)/src --go_out=. simple.proto
```

因为在 proto 中有时会 import  一些别的 proto 文件, 这时就会需要用 `-I` 参数指定protoc 的搜索 import 的 proto 文件夹. 

而可以看到后面 使用 `go-out`进行代码输出, 他会去 调用 protoc-gen-go 这个二进制文件, 这就给我们留了非常大的扩展性, `cpp_out` 用来生成 C++ 代码, `java_out` 生成 Java 代码, `python_out` 生成 PY代码.

 假设我们自己写的插件名字叫 `protoc-gen-helloworld` 那么你只需要 在结尾加上 `helloworld_out` , 那么 protoc 就会去调用你自己写的插件 , go-micro 的 `protoc-gen-micro`  就是用了这个方式.

### 关键字

按照我们的书写顺序依次介绍 会用到的关键字

#### syntax

syntax 用来定义说使用的 protobuf 语法的版本

```protobuf
syntax = "proto3";
```

#### import 

import 用于复用和引用其他 .proto 文件,  而 import 具有两个 关键字 
* weak
  * weak 允许 引入的文件不存在, 也就是在 import 这里不会报错, 不过 如果 后面的使用了不存在的 对象或者结构, 则一样会报错
* public
  * public 通常用于多重引用和阻止多重引用的场景, 其实际作用可以通过下面这一幅图来说明. 图片来自[@hanschen](http://blog.hanschen.site/2017/04/08/protobuf3/)
  * ![](https://raw.githubusercontent.com/shensky711/Pictures/master/2019-9-2-12-36-49.png)
    * 在情景 1 中 my.proto 不能使用  `First.proto`  中引用的 `Second.proto` 文件的内容
    * 在情景2中, my.proto 可以使用 `second.proto`   中的内容

#### package

package 关键字一方面作为 proto 文件的命名空间, 防止 message 类型之间的名字冲突, 同名的 Message 可以通过 package 进行区分.

另一方面也可以用来生成特定语言的 Package 名字, 例如 Java 的Package, 以及 Go 的 Package

#### message

// TODO

#### service

// TODO

#### option

主要的 options 分为四种, 

* File 层级的 option , 较通用的例如 `optimize_for` 等, 需要定义在 最外层空间中, 例如下面这样

  * ```protobuf
    option optimize_for = CODE_SIZE;
    ```

* Message 层级的 option, 需要定义在 Message 中, 例如下面这样

  * ```protobuf
    message HelloWorld {
    	string name = 1;
    	option message_set_wire_format = true;
    	option deprecated = true; // 标示即将弃用
    }
    ```

* Field 层级的 option, 还记得我们前面提到的字段结构吗, Field 层级的options 需要定义在 字段的最后面, 例如下面这样,

  * ```protobuf
    message HelloWorld {
    	string name = 1 [ packed = true, deprecated=true];
    }
    ```

* 最后一种 option 则是 例如 `oneofOptions`,`EnumOptions` 的 options.

关于 `Option` 的详细定义文档, 在可以参考 `github.com/protocalbuffers/protobuf`中的 [descriptor.proto](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/descriptor.proto) 文件. 这个文件中描述了全部预定义的  options . 前面的 四种 options 对应与 descriptor.proto 中的实际 Message Struct 的对应关系如下所示.

* File 层级 >> `FileOptions`
* Message 层级 >> `MessageOptions`
* Field 层级 >> `FieldOptions`
* 最后一种则是, `OneofOptions`,`EnumOptions`,`EnumValueOptions`,`ServiceOptions`,`MethodOptions`



与此同时, 你也可以对 Options 进行自定义, 追加一些自定义的 options 到指定的层级, 如何 自定义如下所示:

```protobuf
import "google/protobuf/descriptor.proto";

extend google.protobuf.MessageOptions {
    string my_option = 51234;
}

message MyMessage {
    option (my_option) = "Hello world!";
}
```

// TODO ~~此外, 我们内部也定义了一些 自定义的 Proto~~

#### 普通字段关键字

首先是 数据类型 定义, 

* 数字类型
  * double
  * float
  * int32
  * int64
  * uint32
  * uint64
  * sint32
  * sint64
* 占用空间固定的数值类型,
  * fixed32
  * fixed64
  * sfixed32
  * sfixed64
  * 在消息序列化的时候大概会有不同? // TODO 待查证, 也欢迎补充
* 布尔类型 布尔类型
* 字符串: `string`
* bytes byte 数组
* messageType 消息类型
* enumType 枚举类型

与各种语言中的数据类型对应关系 请参考 [Google 官方的 文档](https://developers.google.com/protocol-buffers/docs/proto3#scalar)

#### oneof

正如它的名字, one of , 你可以利用这个关键字限制这一组关键字中, 最多只允许一个字段出现.

```protobuf
syntax = "proto3";
package abc;
message OneofMessage {
    oneof test_oneof {
      string name = 4;
      int64 value = 9;
    }
}	
```

另一方面,  因为 proto3 没有办法区分 字段是 设置了还是自动使用了缺省值 (例如 int64 中的0), 甚至你无法判断数据是否有包含这个字段, 因为 protobuf  的 go 实现中, 字段会默认带 `omitempty` 标签, 在字段的值 empty 的时候, 会在传输时省略掉这个值. 

所以你可以通过 oneof 来取得这个功能, 在 protobuf 的 go 实现中, 声明为 oneof 类型的字段, 默认会对应一个 struct ,  当该值未设置的时候 , 接收到的值将会是  nil, 如果已设置, 则会是一个正常值.

另外 oneof 字段不能同时使用 `repeated`

#### map 

map 类型和 go 里的 map 类型基本一致, 与 php 中的 `关联数组`一致, 是一个 KV 键值对. map 类型的格式和上面提到的通常格式略有不同, 这里使用的类 Java 风格的尖括号形式

```protobuf
"map" "<" keyType "," valueType ">" mapName "=" fieldNumber ["[" fieldOptions"]"]
map<int64,string> values = 1;
```

相应的 map 字段也不能同时使用 `repeated`

#### reserved

reserved 应该算是用的比较少的关键字, 他是 一个 Message 层级的关键字,  可以用来声明 当前的 Message 不使用某些字段或者 `FieldNumber`, 也称为保留这些字段. 例子如下.

 另外, 建议声明了 reserved 就不要再定义在 proto 文件中, 否则使用 protoc 编译的时候会报错.  

```protobuf
syntax = "proto3";

package abc;
message AllNormalypes {
  reserved 2, 4 to 6;
  reserved "field14", "field11";
  double field1 = 1;
  // float field2 = 2;
  int32 field3 = 3;
  // int64 field4 = 4;
  uint32 field5 = 5;
  // uint64 field6 = 6;
  sint32 field7 = 7;
  sint64 field8 = 8;
  fixed32 field9 = 9;
  fixed64 field10 = 10;
  // sfixed32 field11 = 11;
  sfixed64 field12 = 12;
  bool field13 = 13;
  // string field14 = 14;
  bytes field15 = 15;
}
```

#### enum

enum 在 go 中没有对应的类型, enum 的中文是 `枚举类型`, 它规定 某个变量只能取几个特定的值, 例子如下所示.

```protobuf
// 规定 Language 字段只能取 "Java" , "PHP" , "Rust" 这三个值
enum Language {
  Java = 0;
  PHP = 1;
  Rust = 1;
}
```

此外, 在proto的定义中, 不允许有 enum 的枚举值相同 也不允许枚举值和 enum 的名字相同, 如下三个例子都会报错.

```protobuf
package example;
// example 1
// Error: "A" is already defined in "example".
enum A {
  A = 0;
  B = 1;
  C = 2;
}
// -----------------------------
// example 2
// Error: "B" is already defined in "example".
enum A {
  B = 0;
  C = 1; 
}

enum B {
  D = 0;
}
// -----------------------------
// example 3
// Error: "C" is already defined in "example".
enum A {
  B = 0;
  C = 1; 
}

enum D {
  C = 0;
  E = 1;
}
```

此外你可以设置 `allow_alias` 这个 option 在 Message 中, 它将允许 FieldNumber 可以重复, 相同 FieldNumber 的字段将会互相作为别名.例如下面这个例子, B和C 互相是别名

```protobuf
enum A {
  option allow_alias = true;
  B = 0;
  C = 0; 
}
```

另外, 枚举值也有较为严格的规定, 第一个枚举值必须为 0, 而且必须定义, 例如下面的 第一个例子 将会报错, 而第二个例子会正常运行

```protobuf
// example 1
enum A {
  B = 1;  // 这一个必须为 0
  C = 0;
}
// --------------
// example 2
enum B {
  C = 0;
  D = 10;
  E = 100;
  F = 1;
}
```

#### 引用其他的 message 作为字段的类型

在下面 SearchResponse 中使用了 Result 作为字段类型, 这种操作较为常见

```protobuf
message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}

message SearchResponse {
  repeated Result results = 1;
}		// ^^^^^^
```

#### 嵌套定义

通常当某个子结构只被父结构使用时, 可以考虑将其作为这样的嵌套定义, 和 go 中 struct 的嵌套定义类似

```protobuf
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

#### unknown

通常我们自己写 .proto 文件时不会用到这个字段,  在proto3 遇到字段解释器无法识别的字段类型时, 会将其标记为 unknown 类型.

#### any

Any 类型允许用户自行处理数据, 不需要经过 proto 定义的类型处理. 一个 Any 类型以 bytes 呈现序列化的消息, 并且包含一个 URL 作为这个类型的唯一标示和元数据.

不过, 为了使用 Any 类型, 你需要引入 `google/protobuf/any.proto`, 例子如下

```protobuf
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
//^^^^^^^^ 这里的 repeated 不能省略, 因为 bytes 必然是以 数组的形式出现,例如 go 中的 []bytes
}
```

### 更新消息类型

有时候你不得不修改正在使用的proto文件，比如为类型增加一个字段，protobuf支持这种修改而不影响已有的服务，不过你需要遵循一定的规则：

- 不要改变已有字段的字段编号
- 当你增加一个新的字段的时候，老系统序列化后的数据依然可以被你的新的格式所解析，只不过你需要处理新加字段的缺省值。 老系统也能解析你信息的值，新加字段只不过被丢弃了
- 字段也可以被移除，但是建议你Reserved这个字段，避免将来会使用这个字段
- int32, uint32, int64, uint64 和 bool类型都是兼容的
- sint32 和 sint64兼容，但是不和其它整数类型兼容
- string 和 bytes兼容，如果 bytes 是合法的UTF-8 bytes的话
- 嵌入类型和bytes兼容，如果bytes包含一个消息的编码版本的话
- fixed32和sfixed32, fixed64和sfixed64
- enum和int32, uint32, int64, uint64格式兼容
- 把单一一个值改变成一个新的oneof类型的一个成员是安全和二进制兼容的。把一组字段变成一个新的oneof字段也是安全的，如果你确保这一组字段最多只会设置一个。把一个字段移动到一个已存在的oneof字段是不安全的

### protobuf Go 实现的扩充类型 

#### wrapper

#### timestamp

#### struct



## Proto 3 的不便与应对 

1. 也是前面提过的 proto3 中没有办法区分字段是设置了空值还是 自动使用了缺省值.
2. 所有的字段都会默认待 `omitempty` 标签, 这个对 go-micro 中 异构系统通过 Api 网关通信场景下不太友好, 在另一个异构系统接收到的值中, 这个字段将会直接缺失.



ref:

> [Protobuf 终极教程](https://colobu.com/2019/10/03/protobuf-ultimate-tutorial-in-go)
>
> [Language Guide (proto3)](https://developers.google.com/protocol-buffers/docs/proto3)
>
> [Protocol Buffers 手册](http://blog.hanschen.site/2017/04/08/protobuf3/)
>
> [proto3默认值与可选项](http://kaelzhang81.github.io/2017/05/15/proto3默认值与可选项/)
>
> [[Protobuf中的Options功能](https://xenojoshua.com/2018/02/protobuf-options/)](https://xenojoshua.com/2018/02/protobuf-options/)
>
> [google/protobuf/descriptor.proto](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/descriptor.proto)