Net的转换器支持自定义实现任何类型转换.

例如请求动作Post指定泛型R, 在转换器NetConverter中将数据转换成R类型返回, 即可实现自定义数据类型转换(支持List等嵌套泛型对象).

> 请注意如果你在转换器里面返回Null. 那么指定的泛型也应当是可空类型, 例如`Post<String?>("api")`

```kotlin
scopeNetLife {
    val userList = Get<List<UserModel>>("list") {
        converter = GsonConverter()
    }.await()
}
```

<br>

具体代码可以阅读Demo

<img src="https://i.loli.net/2021/05/18/yUBmka6AjKsVleP.png" width="300"/>


## 设置转换器
转换器分为全局和单例, 单例可以覆盖全局的转换器. 如果不设置转换器就会采用默认的转换器

=== "全局"
    ```kotlin hl_lines="2"
    NetConfig.init("http://github.com/") {
        setConverter(SerializationConverter())
    }
    ```
=== "单例"
    ```kotlin hl_lines="3"
    scopeNetLife {
        tvFragment.text = Get<String>("api"){
            converter = SerializationConverter()
        }.await()
    }
    ```

## 默认支持类型

默认使用的是: [NetConverter.DEFAULT](https://github.com/liangjingkanji/Net/blob/master/net/src/main/java/com/drake/net/convert/NetConverter.kt)

??? summary "默认转换器源码"
    ```kotlin
    val DEFAULT = object : NetConverter {

        override fun <R> onConvert(
            succeed: Type,
            response: Response
        ): R? {
            return when (succeed) {
                String::class.java -> response.body?.string() as R
                ByteString::class.java -> response.body?.byteString() as R
                ByteArray::class.java -> response.body?.bytes() as R
                Response::class.java -> response as R
                File::class.java -> response.file() as R
                else -> throw ConvertException(
                    response,
                    "The default converter does not support this type"
                )
            }
        }
    }
    ```

| 函数 | 描述 |
|-|-|
| String | 字符串 |
| ByteArray | 字节数组 |
| ByteString | 内部定义的一种字符串对象 |
| Response | 最基础的响应 |
| File | 文件对象, 这种情况其实应当称为`下载文件` |


> 你的业务可能需要参数加密解密或者拼接参数, 请不要尝试封装Post或者Get等请求函数(这不是一个好主意), 自定义拦截器和转换器可以应对任何项目需求


## 自定义转换器

通过实现`NetConverter`接口可以编写自己的逻辑网络请求返回的数据, `NetConvert.DEFAULT`为默认的转换器支持返回File/String/Response等



框架中自带一个`JSONConverter`可以作为参考或者直接使用. 其可以转换JSON数据.

??? summary "JSONConverter 源码"
    ```kotlin
    /**
     * 常见的JSON转换器实现, 如果不满意继承实现自定义的业务逻辑
     *
     * @param success  后端定义为成功状态的错误码值
     * @param code  错误码在JSON中的字段名
     * @param message  错误信息在JSON中的字段名
     */
    abstract class JSONConvert(
        val success: String = "0",
        val code: String = "code",
        val message: String = "msg"
    ) : NetConverter {

        override fun <R> onConvert(succeed: Type, response: Response): R? {
            try {
                return NetConverter.onConvert<R>(succeed, response)
            } catch (e: ConvertException) {
                val code = response.code
                when {
                    code in 200..299 -> { // 请求成功
                        val bodyString = response.body?.string() ?: return null
                        return try {
                            val json = JSONObject(bodyString) // 获取JSON中后端定义的错误码和错误信息
                            if (json.getString(this.code) == success) { // 对比后端自定义错误码
                                bodyString.parseBody<R>(succeed)
                            } else { // 错误码匹配失败, 开始写入错误异常
                                val errorMessage = json.optString(
                                    message,
                                    NetConfig.app.getString(com.drake.net.R.string.no_error_message)
                                )
                                throw ResponseException(response, errorMessage)
                            }
                        } catch (e: JSONException) { // 固定格式JSON分析失败直接解析JSON
                            bodyString.parseBody<R>(succeed)
                        }
                    }
                    code in 400..499 -> throw RequestParamsException(response, code.toString()) // 请求参数错误
                    code >= 500 -> throw ServerResponseException(response, code.toString()) // 服务器异常错误
                    else -> throw ConvertException(response)
                }
            }
        }

        /**
         * 反序列化JSON
         *
         * @param succeed JSON对象的类型
         * @receiver 原始字符串
         */
        abstract fun <R> String.parseBody(succeed: Type): R?
    }

    ```

JSONConvert的核心逻辑

1. 判断服务器的错误码
1. 判断后端自定义的错误码
1. 如果判断发生错误则抛出一个包含错误信息的异常
1. 如果都判断成功则开始解析数据并return数据对象

在转换器中根据需要你可以在这里加上常见的解密数据, token失效跳转登录, 限制多端登录等逻辑. 日志信息输出请阅读: [日志记录器](log-recorder.md)

如果是错误信息建议抛出异常, 就可以在全局异常处理器中统一处理, 请阅读:[全局错误处理](error-handle.md)

<br>

## Json解析库转换器

一般业务我们可以直接继承[JSONConverter](https://github.com/liangjingkanji/Net/blob/master/net/src/main/java/com/drake/net/convert/JSONConvert.kt)
使用自己的JSON解析器解析数据, 自定义需求可以直接实现[NetConverter](https://github.com/liangjingkanji/Net/blob/master/net/src/main/java/com/drake/net/convert/NetConverter.kt)

=== "Kotlin-Serialization"

    ```kotlin
    class SerializationConverter(
        val success: String = "0",
        val code: String = "code",
        val message: String = "msg"
    ) : NetConverter {

        private val jsonDecoder = Json {
            ignoreUnknownKeys = true // JSON和数据模型字段可以不匹配
            coerceInputValues = true // 如果JSON字段是Null则使用默认值
        }

        override fun <R> onConvert(succeed: Type, response: Response): R? {
            try {
                return NetConverter.onConvert<R>(succeed, response)
            } catch (e: ConvertException) {
                val code = response.code
                when {
                    code in 200..299 -> { // 请求成功
                        val bodyString = response.body?.string() ?: return null
                        val kType = response.request.kType() ?: return null
                        return try {
                            val json = JSONObject(bodyString) // 获取JSON中后端定义的错误码和错误信息
                            if (json.getString(this.code) == success) { // 对比后端自定义错误码
                                bodyString.parseBody<R>(kType)
                            } else { // 错误码匹配失败, 开始写入错误异常
                                val errorMessage = json.optString(
                                    message,
                                    NetConfig.app.getString(com.drake.net.R.string.no_error_message)
                                )
                                throw ResponseException(response, errorMessage)
                            }
                        } catch (e: JSONException) { // 固定格式JSON分析失败直接解析JSON
                            bodyString.parseBody<R>(kType)
                        }
                    }
                    code in 400..499 -> throw RequestParamsException(response, code.toString()) // 请求参数错误
                    code >= 500 -> throw ServerResponseException(response, code.toString()) // 服务器异常错误
                    else -> throw ConvertException(response)
                }
            }
        }

        fun <R> String.parseBody(succeed: KType): R? {
            return jsonDecoder.decodeFromString(Json.serializersModule.serializer(succeed), this) as R
        }
    }
    ```

    SerializationConverter和JSONConverter代码差不多
    [Kotlin-Serialization](https://github.com/Kotlin/kotlinx.serialization)

=== "Gson"

    ```kotlin
    class GsonConvert : JSONConvert(code = "code", message = "msg", success = "200") {
        val gson = GsonBuilder().serializeNulls().create()

        override fun <S> String.parseBody(succeed: Type): S? {
            return gson.fromJson(this, succeed)
        }
    }
    ```
    [GSON](https://github.com/google/gson)

=== "Moshi"

    ```kotlin
    class MoshiConvert : JSONConvert(code = "code", message = "msg", success = "200") {
        val moshi = Moshi.Builder().build()

        override fun <S> String.parseBody(succeed: Type): S? {
            return moshi.adapter<S>(succeed).fromJson(this)
        }
    }
    ```
    [Moshi](https://github.com/square/moshi)

=== "FastJson"

    ```kotlin
    class FastJsonConvert : JSONConvert(code = "code", message = "msg", success = "200") {

        override fun <S> String.parseBody(succeed: Type): S? {
            return JSON.parseObject(this, succeed)
        }
    }
    ```
    [FastJson](https://github.com/alibaba/fastjson)

1. 使用转换器时请添加其依赖
2. 推荐使用 `kotlinx.Serialization`, 其可解析[任何泛型](kotlin-serialization.md)
3. 推荐阅读Demo

以上转换器示例是建立在数据结构为以下表格的固定格式下, 如果有特殊的业务可能需要自行修改

| 转换器参数 | 描述 |
|-|-|
| code | 即后端定义的`成功码`字段名 |
| message | 即后端定义的`错误消息`字段名 |
| success | 即`成功码`的值等于指定时才算网络请求成功 |

> 注意解析器(Gson或者Moshi)的解析对象记得定义为类成员, 这样可以不会导致每次解析都要创建一个新的解析对象, 减少内存消耗
<br>