---
layout: post
comments: true
title: "RxJava学习总结"
description: "RxJava学习总结"
category: android
tags: [Android]
---

##  什么是RxJava

###  1. 定义

`RxJava is a Java VM implementation of Reactive Extensions: a library for composing asynchronous and event-based programs by using observable sequences.`

RxJava是JVM的响应式扩展（ReactiveX），它是通过使用可观察的序列将异步和基于事件的程序组合起来的一个库。

###  2. 特点

#### （1）观察者模式

RxJava用到了设计模式中的观察者模式。支持数据或事件序列，允许对序列进行组合，并对线程、同步和并发数据结构进行了抽象。

#### （2）轻量

无依赖库、Jar包小于1M

#### （3）支持多语言

支持Java 6+和Android 2.3+。RxJava设计初衷就是兼容所有JVM语言，目前支持的JVM语言有Groovy,Clojure,JRuby,
Kotlin和Scala。

#### （4）多线程支持

封装了各种并发实现，如threads, pools, event loops, fibers, actors。

###  3. RxJava vs. Java

为了便于大家更好的理解这个新伙伴，学姐总结了RxJava和Java的异同。下面从`关于异步序列`，`数据获取方式`，`数据传递方式`，`增强功能`4个方面来阐述。

#### （1）关于异步序列

通常我们获取一个同步对象，可以这么写`T getData()`；获取一个异步对象，可以这么写`Future<T> getData()`；而获取一个同步序列，可以这么写`Iterable<T> getData()`。那获取一个异步序列呢，Java没有提供相应方法，RxJava填充了这一空白，我们可以这么写`Observable<T> getData()`，关于Observable的相关介绍稍后会有。

#### （2）数据获取方式

Java中如果不使用观察者模式，数据都是主动获取，即**Pull**方式，对于列表数据，也是使用Iterator轮询获取。RxJava由于用到了观察者模式，数据是被动获取，由被观察者向观察者发出通知，即**Push**方式。

#### （3）数据传递方式

对于同步数据操作，Java中可以顺序传递结果，即**operation1 -> operation2 -> operation3**。异步操作通常则需要使用Callback回调，然后在回调中继续后续操作，即**Callback1 -> Callback2 -> Callback3**，可能会存在很多层嵌套。而RxJava同步和异步都是链式调用，即**operation1 -> operation2 -> operation3**，这种做法的好处就是即时再复杂的逻辑都简单明了，不容易出错。

#### （4）增强功能

比观察者模式功能更强大，在onNext()回调方法基础上增加了onCompleted()和OnError()，当事件执行完或执行出错时回调。此外还可以很方便的切换事件生产和消费的线程。事件还可以组合处理。

**说了这么多，然而好像并没有什么用，还是来个例子吧。**

假设需要找出某个本地url列表中的图片本地目录，并且加载对应的图片，展现在UI上。

    new Thread() {
    @Override
    public void run() {
        super.run();
        for (String url : urls) {
            if (url.endsWith(".png")) {
                final Bitmap bitmap = getBitmap(url);
                getActivity().runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        imageView.setImageBitmap(bitmap);
                    }
                });
            }
        }
    }
    }.start();

而使用`RxJava`后，代码是这样的：

    Observable.from(urls)
    .filter(new Func1<String, Boolean>() {
        @Override
        public Boolean call(String url) {
            return url.endsWith(".png");
        }
    })
    .map(new Func1<String, Bitmap>() {
        @Override
        public Bitmap call(String url) {
            return getBitmap(url);
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<Bitmap>() {
        @Override
        public void call(Bitmap bitmap) {
            imageView.addImage(bitmap);
        }
    });
    
## API介绍及使用

由于RxJava内容较多，学姐打算采用**从基础到高级**循序渐进的方式讲解。

RxJava很重要的思想就是观察者模式，对API的介绍也是根据这个模式划分。

###  1. 基础

#### （1）观察者（Observer、Subscriber）

Observer是一个接口，提供了3个方法：`onNext(T t)`, `onError(Throwable e)`, `onCompleted()`。

Subscriber是Observer的子类，`class Subscriber<T> implements Observer<T>, Subscription`。

Subscriber在Observer的基础上有如下扩展：

1. 增加了`onStart()`。这个方法在观察者和被观察者建立订阅关系后，而被观察者向观察者发送消息前调用，主要用于做一些初始化工作，如数据的清零或重置。

2. 增加了`unsubscribe()`。这个方法用于取消订阅，若`isUnsubscribed()`为true，则观察者不能收到被观察者的消息。

创建一个Observer：
       
    Observer<String> observer = new Observer<String>() {
        @Override
        public void onCompleted() {
            Log.d(TAG, "onCompleted");
        }

        @Override
        public void onError(Throwable e) {
            Log.d(TAG, "onError" + e);
        }

        @Override
        public void onNext(String s) {
            Log.d(TAG, "onNext -> " + s);
        }
    };
   
创建一个Subscriber：

    Subscriber<String> subscriber = new Subscriber<String>() {
        @Override
        public void onCompleted() {
            Log.d(TAG, "onCompleted");
        }

        @Override
        public void onError(Throwable e) {
            Log.d(TAG, "onError" + e);
        }

        @Override
        public void onNext(String s) {
            Log.d(TAG, "onNext -> " + s);
        }
    };

#### （2）被观察者（Observable）

Observable决定什么时候触发事件以及触发怎样的事件。常见的3种创建Observable的方式：

1 `Observable.create(Observable.OnSubscribe)`
     
        Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("message 1");
                subscriber.onNext("message 2");
                subscriber.onCompleted();
            }
        });

当`Observable`与`Observer/Subscriber`建立订阅关系的时候，`call()`会被调用。

2 `Observable.just(T...)`

        Observable<String> observable1 = Observable.just("message 1", "message 2");

这个方法可以传1－N个类型相同的参数，和上面的例子等价。最终也会调用`Observeber/Subscriber`的`onNext("message 1")`, `onNext("message 2")`, `onCompleted()`。

3 `Observable.from(T[])`, `Observable.from(Iterable<? extends T>)`

        String[] array = {"message 1", "message 2"};
        Observable<String> observable2 = Observable.from(array);

这个方法可以传数组或Iterable，和上面的例子等价。

#### （3）订阅（Subscription）

订阅关系建立有2种方式：1.`Observable.subscribe(Subscriber)`; 2.`Observable.subscribe(Action)`

`Observable`和`Observer/Subscriber`通过`Observable.subscribe(Subscriber)`建立订阅关系，其内部实现抽取出来如下：

    public Subscription subscribe(Subscriber subscriber) {
        subscriber.onStart();
        onSubscribe.call(subscriber);
        return subscriber;
    }

由源码可知，当订阅关系建立时，首先调用subscriber的`onStart()`方法，此处可进行一些初始化操作，如数据清零或重置。
接着调用`onSubscribe.call(subscriber)`，此处onSubscribe就是创建`Observable`时`Observable.create(OnSubscribe)`传入的`OnSubscribe`参数，说明`Observable`创建时传入的`OnSubscribe`的`call()`回调是在订阅关系建立后调用的。


`Action`这种方式，里面实现也还是用Subscriber进行了包装，本质上就是上面Subscriber的那种方式。只不过根据传入的参数不同回调的方法不同而已，下面代码分别调用Subscriber的`onNext`， `onNext&onError`， `onNext&onError&onCompleted`。

     Subscription subscribe(final Action1<? super T> onNext)
     Subscription subscribe(final Action1<? super T> onNext, final Action1<Throwable> onError)
     Subscription subscribe(final Action1<? super T> onNext, final Action1<Throwable> onError, final Action0 onComplete)

这里学姐想讲讲**Action**，`RxJava`提供了`Action0- Action9`和`ActionN`，这里的数字表示参数的个数分别为0-9和N个。

关于`Subscription`这个接口，这个类提供了两个方法`unsubscribe()`和`isUnsubscribed()`，可以解除订阅关系和判断订阅关系。subscribe()订阅方法的返回值也是`Subscription`。

####（4）场景示例

demo参考github [https://github.com/wangxinghe/tech_explore](https://github.com/wangxinghe/tech_explore)

###  2. 中级

#### （1）线程（Scheduler）

`Scheduler`是`RxJava`的线程调度器，可以指定代码执行的线程。RxJava内置了几种线程：

`AndroidSchedulers.mainThread()` 主线程

`Schedulers.immediate()` 当前线程，即默认`Scheduler`

`Schedulers.newThread()` 启用新线程

`Schedulers.io()` IO线程，内部是一个数量无上限的线程池，可以进行文件、数据库和网络操作。

`Schedulers.computation()` CPU计算用的线程，内部是一个数目固定为CPU核数的线程池，适合于CPU密集型计算，不能操作文件、数据库和网络。

`subscribeOn()`和`observeOn()`可以用来控制代码的执行线程。学姐学习这里的时候，很容易搞混这两个方法分别控制哪部分代码，其实咱们直接跑个demo就明白了。

    Observable.create(new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> subscriber) {
            Log.d(TAG, "OnSubscribe.call Thread -> " + Thread.currentThread().getName());
            subscriber.onNext("message");
        }
    }).subscribeOn(Schedulers.io())
      .observeOn(AndroidSchedulers.mainThread())
      .subscribe(new Subscriber<String>() {
          @Override
          public void onCompleted() {

          }

          @Override
          public void onError(Throwable e) {

          }

          @Override
          public void onNext(String s) {
              Log.d(TAG, "Subscriber.onNext Thread -> " + Thread.currentThread().getName());
          }
      });
      
根据打印出的Log可以得出结论：

subscribeOn()指定OnSubscribe.call()的执行线程，即Observable通知Subscriber的线程；

observeOn()指定Subscriber回调的执行线程，即事件消费的线程。
          
#### （2）变换(map, flatMap)

RxJava提供了一个很**牛逼**的功能，可以对事件或事件序列进行变换，使之转换成不同的事件或事件序列。

有两个常用方法的方法支持变换：`map()`和`flatMap()`。

`map`为一对一变换。可以将一个对象转换成另一个对象，或者将对象数组的每单个对象转换成新的对象数组的每单个对象。

`flatMap()`为一对多变换。可以将一个对象转换成一组对象，或者将对象数组的每单个对象转换成新的对象数组的每单组对象。

以Person为例，一个Person对应一个身份证id，一个Person可以有多个Email。通过`map()`可以将Person转换成id，从而得到一个Person的身份证号码；通过`flatMap()`可以将 Person转换成一组Email，从而得到一个Person的所有Email。

示例如下：

    /**
     * map: Person -> id(String)
     * 打印某个人id
     */
    private void testMap0() {
        Observable.just(getPersonArray()[0])
                .map(new Func1<Person, String>() {
                    @Override
                    public String call(Person person) {
                        return person.id;
                    }
                })
                .subscribe(new Subscriber<String>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(String id) {
                        Log.d(TAG, "id -> " + id);
                    }
                });
    }

    /**
     * map: array Person -> id(String)
     * 打印每个人的id
     */
    private void testMap() {
        Observable.from(getPersonArray())
                .map(new Func1<Person, String>() {
                    @Override
                    public String call(Person person) {
                        return person.id;
                    }
                })
                .subscribe(new Subscriber<String>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(String id) {
                        Log.d(TAG, "id -> " + id);
                    }
                });
    }

    /**
     * flatMap: array Person -> email数组（String[]）
     * 打印每个人的所有email
     */
    private void testFlatMap() {
        Observable.from(getPersonArray())
                .flatMap(new Func1<Person, Observable<Person.Email>>() {
                    @Override
                    public Observable<Person.Email> call(Person person) {
                        Log.d(TAG, "flatMap " + person.id);
                        return Observable.from(person.emails);
                    }
                })
                .subscribe(new Subscriber<Person.Email>() {
                    @Override
                    public void onCompleted() {
                        Log.d(TAG, "onCompleted");
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.d(TAG, "onError " + e.getMessage());
                    }

                    @Override
                    public void onNext(Person.Email email) {
                        Log.d(TAG, "onNext " + email.name);
                    }
                });
    }

###  3. 高级

学姐也是这周才开始学`RxJava`，对于一个全新的知识点，我认为还是以知道怎么使用和一些基础API的基本原理为主，对于更高级的应该是等到API使用较为熟练之后再去学习。因此这里也就不再阐述啦，不能一口吃成一个胖子。

## RxJava与Retrofit组合

`Retrofit`是**Square**公司提供的一个类型安全的Http Client，由于`Retrofit`本身是支持`RxJava`的，因此这两者理所当然搭配使用。

###  场景一

先看下单独使用`Retrofit`进行网络操作的例子：

    public interface GithubAPI {
        @GET("/users/{user}")
        public void getUserInfo(@Path("user") String user, Callback<UserInfo> callback);
    ｝

    private void fetchUserInfo() {
        String username = mEditText.getText().toString();
        getGithubAPI()
                .getUserInfo(username, new Callback<UserInfo>() {
                    @Override
                    public void success(UserInfo userInfo, Response response) {
                        mTextView.setText(userInfo.email);
                    }

                    @Override
                    public void failure(RetrofitError error) {
                        mTextView.setText(error.getMessage());
                    }
                });

    }
    
单独使用`Retrofit`是需要有回调的，如果逻辑稍微在复杂点，可能又要在Callback里做很多事情，代码维护会很费劲。

下面看下`Retrofit`搭配`RxJava`使用的例子：

    public interface GithubAPI {
        @GET("/users/{user}")
        public Observable<UserInfo> getUserInfo(@Path("user") String user);
    ｝

    private void fetchUserInfoRx() {
        String username = mEditText.getText().toString();
        getGithubAPI()
                .getUserInfo(username);
                .subscribe(new Observer<UserInfo>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {
                        mTextView.setText(e.getMessage());
                    }

                    @Override
                    public void onNext(UserInfo userInfo) {
                        mTextView.setText(userInfo.email);
                    }
                });
    }
    
###  场景二

下面再看下多个操作的情况，比如username需要根据网络操作获取，然后才能通过username获取用户信息。

则`Retrofit`的代码是这样的：

    public interface GithubAPI {
        @GET("/username")
        public void getUserName(Callback<String> callback);
    
        @GET("/users/{user}")
        public void getUserInfo(@Path("user") String user, Callback<UserInfo> callback);
    ｝

    private void fetchUserInfo() {
        String username = mEditText.getText().toString();
        getGithubAPI()
                .getUserName(new Callback<String>() {//获取username
                    @Override
                    public void success(String username, Response response) {
                        /获取UserInfo
                        getUserInfo(username, new Callback<UserInfo>() {
                            @Override
                            public void success(UserInfo userInfo, Response response) {
                                mTextView.setText(userInfo.email);
                            }

                            @Override
                            public void failure(RetrofitError error) {
                                mTextView.setText(error.getMessage());
                            }
                        })
                    }               
                });
    }

而`Retrofit`搭配`RxJava`使用的代码是这样的：

    public interface GithubAPI {
        @GET("/username")
        public Observable<String> getUserName();

        @GET("/users/{user}")
        public Observable<UserInfo> getUserInfo(@Path("user") String user);
    ｝

    private void fetchUserInfoRx() {
        String username = mEditText.getText().toString();
        getGithubAPI()
                .getUserName()    //获取username
                .flatMap(new Func1<String, Observable<UserInfo>>() {
                    @Override
                    public Observable<UserInfo> onNext(String username) {
                        //获取UserInfo
                        return getGithubAPI().getUserInfo(username);
                    })
                }
                .subscribe(new Observer<UserInfo>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {
                        mTextView.setText(e.getMessage());
                    }

                    @Override
                    public void onNext(UserInfo userInfo) {
                        mTextView.setText(userInfo.email);
                    }
                });
    }

有没有发现，使用`RxJava`结构更清晰明了。

补充下，使用`Retrofit`和`RxJava`是需要添加依赖的：

    compile 'io.reactivex:rxjava:1.1.0'
    compile 'io.reactivex:rxandroid:1.1.0'
    compile 'com.jakewharton.rxbinding:rxbinding:0.4.0'
    compile 'com.squareup.retrofit:retrofit:1.9.0'

示例demo放到github上了，[https://github.com/wangxinghe/tech_explore](https://github.com/wangxinghe/tech_explore)

## 总结

1.RxJava最大特点是链式调用，使异步逻辑结构更清晰明了

2.观察者模式：Obverable(被观察者), Observer/Subscriber(观察者), Subscription(订阅)

3.真的很感谢[扔物线](https://github.com/rengwuxian)，拜读他的文章学习的`RxJava`
 
## 参考文档

1.[http://gank.io/post/560e15be2dca930e00da1083](http://gank.io/post/560e15be2dca930e00da1083)

2.[http://codethink.me/2015/05/09/intro-of-rxjava/](http://codethink.me/2015/05/09/intro-of-rxjava/)

3.[https://github.com/ReactiveX/RxJava](https://github.com/ReactiveX/RxJava)

4.[http://reactivex.io/intro.html](http://reactivex.io/intro.html)

