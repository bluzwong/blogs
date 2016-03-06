title: 使用Kotlin遇到的一些问题
date: 2016-02-22 22:57:16
tags: android kotlin
---
## 前言
[Kotlin](https://github.com/JetBrains/kotlin) 正式版已经发布了，优点大致就是解决了Java的一些痛点，写起来很Exciting。我来说说用Kotlin遇到的问题。  

## Apt(Annotation processing tool) 并不总是起效
Apt 现在可谓是不可缺少的一个工具，有非常多的库钦定了Apt。~~有人问，“不是说用注解会影响运行效率吗？”
我可以回答一句 “无可奉告”，但是你们又不高兴，我怎么办？我讲的意思是，注解不会影响运行效率，反射才会。我就明确告诉你这一点。~~ 有来自西方代码工作者 JakeWharton 的 [ButterKnife](https://github.com/JakeWharton/butterknife), 有来自 Google 官方的 [Data Dinding](http://developer.android.com/intl/zh-cn/tools/data-binding/guide.html) (http://www.jianshu.com/p/b1df61a4df77) 等。  
> Data Binding 内部也使用了Kotlin

Kotlin 呢，其实也有类似的工具， 叫 [Kapt](http://blog.jetbrains.com/kotlin/2015/05/kapt-annotation-processing-for-kotlin/) 但是在 `Kotlin 1.0.0 RC` 之前的版本都是基本不太能用，而之后的版本虽然可以用了，但是总是出现个别注解无法解析的问题。后来发现了出问题的规律：被注解的属性或者字段如果存在名字相同的，就算是其他文件里，也会只有其中的一个可以正常解析。比如使用ButterKnife注入View：
```kotlin
// @ MainActivity.kt
@Bind(R.id.btn)
lateinit var btn: TextView
```
```kotlin
// @ AnotherActivity.kt
@Bind(R.id.btn)
lateinit var btn: TextView
```
这种情况下就只有其中的一个可以被解析了。不过，既然发现了这个规律，也能适当修改来规避这个bug了。
```kotlin
// @ MainActivity.kt
@Bind(R.id.btn)
lateinit var btn_main: TextView
```
```kotlin
// @ AnotherActivity.kt
@Bind(R.id.btn)
lateinit var btn_another: TextView
```
如此，两个View都注入成功。  

这个问题也被西方的一些代码工作者提交了，[KT-9183](https://youtrack.jetbrains.com/issue/KT-9183)，希望能够早日被解决。

### 另外
要正常使用Kapt，除了配置好 Kotlin 依赖外，需要做一些额外的处理：
```groovy
// @ app/build.gradle
...
kapt {
    generateStubs = true
}
dependencies {
    ...
    // apt 'com.jakewharton:butterknife:7.0.1'
    kapt 'com.jakewharton:butterknife:7.0.1'
    ...
}
...
```
`generateStubs = true` 会先把Kotlin的代码生成 `.class` stubs，以处理注解。还有就是要把 `apt` 改为 `kapt`， kapt同样也会对java代码做注解处理，而且不会有上述的问题，这点请放心。 

在使用注解的地方：
```kotlin
@Bind(R.id.btn)
lateinit var btn_another: TextView

@AFieldAnnotation @JvmField
var foo = 0
```
`lateinit` 关键字 和 `@JvmField` 注解 会把被修饰的属性变为普通的Java字段，失去Kotlin赋予属性的一些特性。但是为了要注解字段，做这些牺牲也是值得的。

## Data Binding 无法和 Kotlin 一起使用
不是说 Kotlin 可以 Kapt 吗？Data Binding 内部也用了 Kotlin ，怎么会不能一起使用？  
其实问题就出在 Data Binding 内部也用了 Kotlin。



在 Data Binding Compiler 的 gradle `.pom` 中发现
```xml
<dependency>
    <groupId>org.jetbrains.kotlin</groupId>
    <artifactId>kotlin-stdlib</artifactId>
    <version>1.0.0-beta-4584</version>
    <scope>runtime</scope>
</dependency>
```
![kotlin in data binding](http://7xr14l.com1.z0.glb.clouddn.com/kotlinindatabinding.jpg)
点开文件
![kotlindatabindingnotwarking](http://7xr14l.com1.z0.glb.clouddn.com/kotlindatabindingerror.jpg)
编译提示这个错误
```
Error:cannot generate view binders java.lang.NoSuchMethodError: kotlin.text.StringsKt.trim(Ljava/lang/String;)Ljava/lang/String;
```
唉，等 Google 更新Data Binding的Kotlin版本吧，或者索性移除Kotlin的依赖。每次Kotlin版本更新的时候， 用Kotlin写过第三方库的人懂的都懂...
这个问题也被人反馈到了 [KT-8007](https://youtrack.jetbrains.com/issue/KT-8007) 和 [AOSP](https://code.google.com/p/android/issues/detail?id=201346)。

## Kotlin 方法的引用
Kotlin 目前只有 TopLevel 的方法可以使用方法引用
```kotlin
fun fooAtTopLevel() : Int {
    return 0
}

class FunctionQuestion {
    
    fun fooAtClass() : Int {
        return 0
    }
    
    fun bar(f: () -> Int) {
        f()
    }

    fun fooAsFunctionReference() {
        val fooAtLocal: () -> Int = {
            1
        }
        
        bar(::fooAtTopLevel) // OK
        bar(::fooAtClass) // ERROR !
        bar(this::fooAtClass) // try syntax likes java, but still ERROR !
        bar(fooAtLocal) // OK
    }
}
```
这点还是挺遗憾的，不过 Kotlin 的开发人员说这个特性已经是未来的一个计划了
<http://stackoverflow.com/questions/33616464/function-references-and-lambdas>
<http://stackoverflow.com/questions/32598320/kotlin-method-reference-not-working>

## 还有关于 Anko 的问题
[Anko](https://github.com/Kotlin/anko) 确实写起来超爽
```kotlin
// 传统的代码中生成视图，Kotlin语法，如果用Java会更加冗长
val act = this
val layout = LinearLayout(act)
layout.orientation = LinearLayout.VERTICAL
val name = EditText(act)
val button = Button(act)
button.text = "Say Hello"
button.setOnClickListener {
    Toast.makeText(act, "Hello, ${name.text}!", Toast.LENGTH_SHORT).show()
}
layout.addView(name)
layout.addView(button)
```

```kotlin
// 同样效果的Anko版本
verticalLayout {
    val name = editText()
    button("Say Hello") {
        onClick { toast("Hello, ${name.text}!") }
    }
}
```

但是，在 xml 中的自定义属性
```xml
<View
    xmlns:app="http://schemas.android.com/apk/res-auto"
    ...
    app:layout_collapseMode="parallax"
    ... />
```
该怎么办呢？目前还没有找到处理的法。只能割爱暂时放弃 Anko，等JetBrains钦定了，再去产生。

## Kotlin
总体来说，这些小问题瑕不掩瑜，Kotlin 是一个用起来很愉悦的语言，大家对 Java 诟病的地方，很多都在 Kotlin 中被缓解， 而且和 Java 不说 100% 兼容，至少 90% 还是有的，这点还是很重要，有 Java 做背书，有时候 Kotlin 不能用的地方， 换 Java 就好了嘛。  
一个语言的命运啊，当然要靠自身的奋斗，但也有考虑到历史的进程。Kotlin他爹 JetBrains 怎么也想不到啊，Kotlin 一个在 Jvm 上跑的好好的，怎么就被 Google 用到 Data Binding 了呢？
不管 Kotlin 现在还有什么问题，至少 Google 也是已经把 Kotlin 用到了自己的库中。这是不是给人一种硬点的感觉？

# 我主要的就是这几件事情
# 很惭愧，就做了一点微小的工作。
# 蟹蟹大家！