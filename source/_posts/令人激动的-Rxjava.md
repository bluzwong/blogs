title: 令人激动的 Rxjava
date: 2016-02-25 00:31:51
tags: Rxjava

---
*你觉得用 Rxjava 好不好啊？*
*- 吼啊！当然啦！*  

## 前言
[Rxjava](https://github.com/ReactiveX/RxJava) 到底有什么用呢？一句话的定义:
`Reactive Extensions for the JVM – a library for composing asynchronous and event-based programs using observable sequences for the Java VM.`。  
`处理基于事件的异步问题。`  
让我们通过一个情形来看看如何理解这个定义。

## 情形
假设现在有三个运行比较慢的方法，他们可能是网络接口、也可能是一些复杂的运算，总之运行这些方法比较慢，在android中会阻塞主线程。抽象出来，大概是这样：

```java
    public static int slowGetA() throws InterruptedException {
        /// ... bala bala
        Thread.sleep(200);
        return new Random().nextInt(5);
    }

    public static int slowGetB() throws InterruptedException {
        /// ... bala bala
        Thread.sleep(200);
        return new Random().nextInt(100);
    }

    public static int slowAdd(int a, int b) throws InterruptedException {
        /// ... bala bala
        Thread.sleep(200);
        return a + b;
    }
```

`slowGetA()` 和 `slowGetB()` 不依赖于其他数据，可以直接拿到返回值。`slowAdd` 需要之前两个方法的返回值作为参数，再去得到结果。这样的情形其实是挺常见的，比如先登录请求，获取`tokon` 或者其他一些东西，然后在后面的其他请求带上 `token`。

## 常规实现
`Excuse me?` 这有什么难的？
先定义回调接口：
```java
    // 情形简化需要，处理异常情况等省略
    interface FooListener {
        void onComplete(int result);
    }
```
再把这几个方法封装成异步回调的方式:
```java
    // 情形简化需要，这里只做简单封装，类似于一些网络请求库的异步回调
    public static void asyncGetA(final FooListener listener) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                int a = -1;
                try {
                    a = slowGetA();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                listener.onComplete(a);
            }
        }).start();
    }

    public static void asyncGetB(final FooListener listener) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                int a = -1;
                try {
                    a = slowGetB();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                listener.onComplete(a);
            }
        }).start();
    }

    public static void asyncAdd(final int a, final int b, final FooListener listener) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                int result = -1;
                try {
                    result = slowAdd(a, b);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                listener.onComplete(result);
            }
        }).start();
    }
```
最后，去把他们都组合起来：
```java
    // 由于情形简化需要以及笔者能力有限，采取了如下的实现方式。
    public void onClick(View view) {
            final CountDownLatch latch = new CountDownLatch(2);
            final int[] a = new int[1];
            final int[] b = new int[1];
            asyncGetA(new FooListener() {
                @Override
                public void onComplete(int _a) {
                    // 获取到了第一个返回值
                    // CountDownLatch 计数器减一
                    a[0] = _a;
                    latch.countDown();
                }
            });
            asyncGetB(new FooListener() {
                @Override
                public void onComplete(int _b) {
                    // 获取到了第二个返回值
                    // CountDownLatch 计数器减一
                    b[0] = _b;
                    latch.countDown();
                }
            });
            new Thread(new Runnable() {
                @Override
                public void run() {

                    try {
                        // 开启新的一个线程等待计数器减为0，也就是等待两个返回值都获取完成。
                        latch.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    // 把上面获得的两个返回值作为参数，请求最后的结果。
                    // 这里非主线程，也可以直接使用阻塞的同步方法
                    asyncAdd(a[0], b[0], new FooListener() {
                        @Override
                        public void onComplete(final int result) {
                            runOnUiThread(new Runnable() {
                                @Override
                                public void run() {
                                    // 拿到了最终的结果，在主线程中更新UI
                                    tvNormal.setText("result is : " + result);
                                }
                            });
                        }
                    });
                }
            }).start();

        }
```
蛤蛤 `Too Simple!`，这不就好了嘛！

## 慢着
这个缩进让我想起了某个 [需要带刻度尺写代码的语言](https://www.python.org/)，这一层嵌套着一层的结构，看着有点不怎么舒服。如果后面还要再去请求其他的呢？其中有些接口出现了异常呢？有些方法需要保存或者读取本地缓存呢？如果需要取消掉这些请求呢。。。。  
OMG，Callback Hell!

## Rxjava 实现！
来看看Rxjava是如何优雅的处理这种情形。
第一步，同样的，先把这三个会阻塞的方法进行封装，包装成 Rxjava 的 `Observable` 。
```java
    // 情形简化需要，只做简单封装
    public static Observable<Integer> rxGetA() {
        return Observable.defer(new Func0<Observable<Integer>>() {
            @Override
            public Observable<Integer> call() {
                int result = -1;
                try {
                    result = slowGetA();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return Observable.just(result);
            }
        });
    }

    public static Observable<Integer> rxGetB() {
        return Observable.defer(new Func0<Observable<Integer>>() {
            @Override
            public Observable<Integer> call() {
                int result = -1;
                try {
                    result = slowGetB();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return Observable.just(result);
            }
        });
    }

    public static Observable<Integer> rxAdd(int a, int b) {
        return Observable.defer(new Func0<Observable<Integer>>() {
            @Override
            public Observable<Integer> call() {
                int result = -1;
                try {
                    result = slowAdd(a, b);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return Observable.just(result);
            }
        });
    }
```
定义一个中间类，用于持有数据
```java
class AnB {
    public AnB(int a, int b) {
        this.a = a;
        this.b = b;
    }

    int a;
    int b;
}
```

组合到一起
```java
    public void onClick(View view) {
        // 将两个 Observable 打包到一起。两个 Observable 都执行完成后，
        // 在第三个参数的匿名类中 return 返回值，类似于 `AsyncTask` 的 `doInBackground`
        Observable.zip(rxGetA(), rxGetB(),
                new Func2<Integer, Integer, AnB>() {
                    @Override
                    public AnB call(Integer a, Integer b) {
                        // 用中间类来持有rxGetA()和rxGetB()的结果
                        // 如果有类似 Python的Tuple 就方便多了。。
                        return new AnB(a, b);
                    }
                })
                // 将打包的数据进一步处理
                .flatMap(new Func1<AnB, Observable<Integer>>() {
                    @Override
                    public Observable<Integer> call(AnB anB) {
                        // 从中间类取出两个参数，请求第三个方法获取最后结果
                    return rxAdd(anB.a, anB.b);
                    }
                })
                // 在IO线程进行订阅，也就是上面这些操作都在IO线程中运行，所以不会阻塞主线程
                .subscribeOn(Schedulers.io())
                // 在android主线程观察，也就是下面紧接着的一部操作会在主线程运行，所以可以操作UI
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<Integer>() {
                    @Override
                    public void call(Integer result) {
                        // 操作UI
                        tvNormal.setText("result is : " + result);
                       }
                });
    }
```
*"看起来一点也不比上面的方法短，这特么到底哪里好了？"*
大熊弟啊，代码高不高、宽不宽暂且不说。用Rxjava逻辑上简单了一个数量级。你看，嵌套地狱不见了！所有的操作都是线性的，一排往下走，类似于写同步调用那般清晰。
*"我不管，这个代码太多。"*
大熊弟妮别急嘛。。来加上lambda表达式看看：
```java
    // 使用了java8的lambda表达式，由于android原生只支持java7
    // 所以需要使用 retrolambda 插件实现对lambda的支持
    // 当然啦，更好的解决办法是 原生支持lambda，且和java兼容的Kotlin
    public void onClick(View view) {
        Observable.zip(rxGetA(), rxGetB(),
                AnB::new)
                .flatMap(anB -> rxAdd(anB.a, anB.b))
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(result -> tvRxjava.setText("result is : " + result));
    }
```
这样呢？

## Rxjava 大法好

笔者只是展示了Rxjava庞大操作符的冰山一角，Rxjava的操作符简直无所不能，包括错误处理、过滤、转换等。

参考了这些前辈的贡献：
[ReactiveX 中文文档](https://github.com/mcxiaoke/RxDocs) *(Rxjava 是ReactiveX 在java的实现)*
[RxJava Essentials 中文翻译版](https://github.com/yuxingxin/RxJava-Essentials-CN)
[深入浅出RxJava](http://blog.csdn.net/lzyzsd/article/details/41833541)
[给 Android 开发者的 RxJava 详解](https://gank.io/post/560e15be2dca930e00da1083)

你问我支不支持 Rxjava，我说支持，我就明确告诉你这一点！
用Rxjava也要按照基本法，按照使用的法。当然，我们的决定权也是很重要的。

# 很惭愧，只做了一些微小的工作。
# 蟹蟹大家！
# [所用Demo](https://github.com/bluzwong/excetingrxjava)