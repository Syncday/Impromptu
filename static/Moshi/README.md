
## Moshi转换Json中空对象{}的修复
**Exception:** com.squareup.moshi.JsonDataException: Required value 'xxx' missing at xxx

先说明一下出现问题的json数据，如下
```
{
    "name":"Jack",
    "age":20,
    "address":{}
}
```

kotlin对象如下
```kotlin
data class Address(val country:String,val street:String)

data class User(val name:String,val age:Int,val address:Address?)
```

按照正常情况，官方的KotlinJsonAdapterFactory()应该能转换成功但是实际情况却抛出了异常。

我们想要的效果不过是让?标识的对象为空即可。

一个取巧的方法就是给每个变量添加?标识。但这不是一个好方法，为了解决这个问题，那么需要从根本上去处理。

对其实现代码（KotlinJsonAdapterFactory.kt）进行如下修改
```kotlin
internal class FixKotlinJsonAdapter<T>(
            val constructor: KFunction<T>,
            val allBindings: List<Binding<T, Any?>?>,
            val nonIgnoredBindings: List<Binding<T, Any?>>,
            val options: JsonReader.Options
        ) : JsonAdapter<T>() {

            override fun fromJson(reader: JsonReader): T {
                //省略代码...

               //记录json对象为空的属性
                var nullParameters = 0
                // Confirm all parameters are present, optional, or nullable.
                var isFullInitialized = allBindings.size == constructorSize
                for (i in 0 until constructorSize) {
                    if (values[i] === ABSENT_VALUE) {
                        when {
                            constructor.parameters[i].isOptional -> isFullInitialized = false
                            constructor.parameters[i].type.isMarkedNullable ->{
                                values[i] = null // Replace absent with null.
                            }
                            //判断所有的属性是否都为空
                            (i == nullParameters) -> {
                                isFullInitialized = false
                                values[i] = null
                            }
                            else -> throw Util.missingProperty(
                                constructor.parameters[i].name,
                                allBindings[i]?.jsonName,
                                reader
                            )
                        }
                        nullParameters++
                    }
                }

                if(nullParameters==constructorSize){
                   // No instance needs to be created
                   return null
                }

                //省略代码...
            }

        }
```
_关键字_: Moshi、KotlinJsonAdapter、Kotlin、null safety、Json 
