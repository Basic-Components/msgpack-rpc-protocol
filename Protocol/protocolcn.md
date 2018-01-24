# MESSAGE-PACK-RPC

+ 起始时间:2018-1-22
+ 版本:0.1
+ 作者:hsz

## 概述

MESSAGE-PACK-RPC是一个轻量级的无状态远程过程调用(RPC)应用层协议.它参考自JSON-RPC 2.0协议,
但它是独立的也不向JSON-RPC协议兼容.

本规范主要定义了一些数据结构及其相关的处理规则.它允许运行在基于tcp协议的消息传输环境中,
使用messagepack作为序列化和反序列化数据格式进行通信.


与json-rpc不同,message-pack-rpc的个请求形式可以有

+ 应答模式,一问一答. 类似函数调用
+ 服务端在获取请求后向客户端进行推送. 类似于调用一个生成器
+ 客户端向服务端推送流. 类似函数的参数是一个生成器
+ 客户端向服务端推送流,而服务端的应答也是流,类似可迭代对象的操作


其他的特性包括:

+ 服务器端对连接建立的客户端可以有权限检验,也就是说可以设定口令

+ 客户端主动关闭连接,服务端可以设置过期时间主动断开连接,默认为180秒.

+ 允许设置客户端类型检查默认为True

+ 允许客户端设定心跳以维持连接不会过期断开.默认为False

+ 支持使用json替代message-pack,该模式为DEBUG模式,默认不使用DEBUG模式

+ 可以在通信中进行使用`bz2`或者`zlib`或者`lzma`进行数据压缩.

+ 协议不支持批量调用.

## 数据形式约定

数据形式约定分为如下几个层次:


+ 传输数据类型和关键字格式约定

JSON可以表示四个基本类型(String、Numbers、Booleans和Null)和两个结构化类型(Objects和Arrays).而我们的message-pack也是以这6种基本类型为基础.

我们规定rpc注册的函数的参数和返回值也只能在这6种类型.且要求参数和返回值中的Object不允许嵌套

同时规定必须使用英文大写单词标识协议用到的字段,而协议预设的函数名也必须是大写.

+ 可以用于传输的对象约定

我们约定合法的对象包括:

    + 自描述应答对象
    + 请求对象
    + 响应对象
    + 异常/错误对象
    + 结果对象

+ 连接建立时服务端的自描述应答格式约定

成功建立连接时回调函数会返回一个message-pack字节序列指明本rpc的基本信息,其形式为:

```json
{
    "MPRPC":"0.1",// string 协议版本号
    "VERSION":"0.0.1",//string 服务的版本用于在客户端检验
    "DESC":"xxxx",// string 描述服务
    "DEBUG":true,// bool 是否使用debug模式,也就是传递的是json还是msgpack
    "COMPRESSION":null,// enum 压缩算法,可选的有`bz2`,`zlib`,`lzma`和null
    "TIMEOUT":180,//number 设置的过期时间,设为0表示不设置过期时间
    "FUNCTION_LIST":[
        {
            "METHOD":"get",
            "ANNOTATIONS":{"a":'int','return':null} //类型检测用的函数声明
        },
        ...
        
    ]// array 可被调用的函数名和对应签名
}
```

+ 客户端接收到错误后的行为

    客户端接收到错误后会先关闭连接然后抛出对应异常

+ 由客户端关闭连接后服务端的行为

    客户端关闭连接属于正常行为,没有额外动作

+ 请求对象约定

    ```json
    {
        "MPRPC":"0.1",// string 协议版本号
        "ID":xxxx,//string 任务id
        "METHOD": xxx,//接收到要执行的函数名
        "ARGS":xxx //(OPTION) array or object  接收到函数调用的参数,只允许为列表或者键值对的形式
    }
    ```

+ 响应对象约定

    当发起一个rpc调用时，除通知之外，服务端都必须回复响应。响应表示为一个JSON对象，使用以下成员：

    ```json
    {
        "MPRPC":"0.1",// str 协议版本号
        "CODE":200,// number 响应码,响应码反应服务器状态,
        "MESSAGE": {} // object 对象为结果对象或者异常/错误对象
    }
    ```

    响应码含义表(参考自http协议)

    code|对应错误|意义
    ---|---|--
    100|---|表示初始的请求已经接受,客户应当继续发送请求的其余部分.
    200|---|表示执行正常,返回的结果结束
    202|---|表示已经接受请求,返回为一个流,且还未结束
    206|---|返回为一个流,且结束了
    300|ExpireWarning|即将过期的函数执行正常
    302|ExpireWarning|即将过期的函数,表示已经接受请求,返回为一个流,且还未结束
    306|ExpireWarning|即将过期的函数,返回为一个流,且结束了
    400|RequestError|请求错误
    401|LoginError|登录失败(口令有误)
    403|AuthError|口令权限无法访问对应函数
    404|WrongMethodError|未找到对应的函数名
    470|ParamError|请求的参数与签名不符
    500|RpcException|服务器异常
    502|RequirementException|服务器的依赖服务异常,
    503|RpcUnavailableException|服务器不可用异常,一般是在维护
    504|TimeoutException|服务器连接超时异常


    + 警告/错误异常对象约定

    ```json
    {
        ID: xxx,// string 任务id
        EXCEPTION: xxx// string  服务器端错误/异常类型名
        MESSAGE: xxxx,// string or array or object 错误描述文字
        DATA: {//(OPTION) 错误的附加信息,由服务器决定,下面的是参考字段
            METHOD: //(OPTION)接收到要执行的函数名
            ARGS: //(OPTION)接收到函数调用的参数
            FRAME: //(OPTION)错误的帧信息
        } 
    }
    ```

    + 结果对象约定

    ```json
    {
        ID: xxx,// string 任务id
        RESULT: xxx// any  返回的结果
    }
    ```

### 服务器固有方法约定

+ METHOD_HELP(method:str)->None

    返回指定函数的说明文档


+ METHOD_SIGNATURE(method:str)->obj

    返回指定函数的签名


+ METHODS_LIST()->list[dict]

    返回对外的函数接口


### 客户端可设置参数:

+ host  主机
+ port  端口
+ auth  验证信息,默认为None
+ remote_version 远程服务的版本号,默认为None
+ heart_beat 默认为False

### 服务器可设置参数:

+ DEBUG 是否使用debug模式,默认为False
+ AUTH 验证信息,默认为None
+ COMPRESSION 使用什么进行压缩,默认为None
+ VERSION 服务的版本用于在客户端检验默认为None
+ DESC 描述服务的字符串,默认None
+ TIMEOUT 多久没有请求就关闭连接默认为None
+ MPRPC 协议版本号,默认为最新

## 负载均衡器

可以实现一个专用的负载均衡服务器专门用于分发任务和维护连接


