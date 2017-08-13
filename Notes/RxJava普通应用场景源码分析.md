# RxJava普通应用场景源码分析

## 响应式编程（reactive programming）

想要介绍rxjava，绝对绕不开的就是响应式编程，因为rxjava本身就是响应式编程的一个第三方库。因为我本身对这个编程思想理解的深度也不够，所以在此我不想过多介绍这个编程思想。但是还是分享一篇文章给大家：

[The introduction to Reactive Programming you've been missing](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)

中文版的可以戳这里：

[那些年我们错过的响应式编程](https://github.com/hehonghui/android-tech-frontier/tree/master/androidweekly/%E9%82%A3%E4%BA%9B%E5%B9%B4%E6%88%91%E4%BB%AC%E9%94%99%E8%BF%87%E7%9A%84%E5%93%8D%E5%BA%94%E5%BC%8F%E7%BC%96%E7%A8%8B)

文章中称：*响应式编程就是与异步数据流交互的编程范式*，一个数据流是一个按时间排序的即将发生的事件(Ongoing events ordered in time)的序列。在响应式编程中，数据流应该有三种不同的事件，分别是包含某个值的事件，完成事件和错误事件。引用文中图片：

![事件流](https://camo.githubusercontent.com/bec8630b9f5e74a42f4096ba9fcfe4801f243deb/687474703a2f2f696d672e6d792e6373646e2e6e65742f75706c6f6164732f3230313530342f31332f313432383931343335395f323135302e706e67)

从图中可以看出，它是用点击事件举例，但是其实任何事件都可以是一组数据流：点击、输入、网络请求等等。而在数据流将事件发送出来后，它会调用各自的事件处理函数，所以说它是异步的。在数据发送之前我们可以对数据进行一系列的处理操作。但在分析过rxjava的源码之后你会发现，其实这些处理操作也是作为订阅者先接受到发射出来的数据流，处理之后，再从自己这里发送出新的数据流。
为了理解这个流程，还是用文中的给的一张图给大家看，图中依旧是点击事件数据流：

![事件流](https://camo.githubusercontent.com/b3d40bf50d5a25d15e054f2b29a83328d1531710/687474703a2f2f696d672e6d792e6373646e2e6e65742f75706c6f6164732f3230313530342f31332f313432383931343336305f363432392e706e67)

图中先是接收点击事件，`buffer`对连续发生在250ms内的点击事件映射为一个list，长度就是250ms内的点击次数。`map`接着将接收到的数据流映射为int数据流，大小为对应list的长度。`filter`将int值小于2的顾虑掉，只发射出大于等于2的int数据流。

##为什么我们需要响应式编程？
先放一段名人名言,这是Jake Wharton在一个[分享](https://news.realm.io/cn/news/gotocph-jake-wharton-exploring-rxjava2-android/)中说的：

>Why is Reactive suddenly becoming something that you hear a lot of people talking about? Unless you can model the entire system of your app in a synchronous fashion, having a single asynchronous source will ultimately break the traditional imperative style of programming we’re used to. Not “break” in the sense that it stops working, but in the sense that it pushes the complexity onto you, and you start losing the things that imperative programming is really good for.


>为什么突然之间，您听见大家都在谈论起响应式编程 (Reactive) 了呢？因为除非您可以完全用同步操作来编写整个应用，否则的话，应用中或多或少都会有异步源的存在，而这终将打破我们习惯的传统命令式编程风格。所谓的「打破」，并不是说命令式编程将被废除，而是说在某种意义上，它导致编程的复杂性极度增大，这个时候您就不会觉得命令式编程仍然是一个不错的选择了


作为一个android开发者，在以前如果想实现异步数据流的最简单方式就是Handler+Thread或者其他的方式，除非完全用同步操作来编写整个应用，否则的话，应用中或多或少都会有异步源的存在。个人觉得这是非常不直观的，不符合人类的思考方式，我们总是思考从什么地方获取数据，中间做了哪些处理，最后如何展示这些数据，这是一个很明晰的流程，代码中应该从上至下，一步接着一步阐述清楚。而以前的代码书写方式很难做到这一点。
而利用rxjava这样的第三库书写这样的逻辑的时候能够非常简洁，比如：


```java
new Thread() {
    @Override
    public void run() {
        super.run();
        //do something...
        final String s = "hello world!";
        getActivity().runOnUiThread(new Runnable() {
            @Override
            public void run() {
                if (s != null) {
                    txtView.setText(s);
                }
            }
        });
    }
}.start();
```

利用rxjava可以写成：

```java
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        //do something...
        subscriber.onNext("hello world!");
        subscriber.onCompleted();
    }
})
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                if (s != null) {
                    txtView.setText(s);
                }
            }
        });

```
乍一看，代码量确实还增加了，但是可以明显看到rxjava写的逻辑上更加简洁，是一条从上到下的链式调用。如果别的程序员看到这样的代码的时候，能够很快理解整个逻辑。

# 最常见的一段代码
```java
Observable.just("hello world!")
        .map(String::length)
        .subscribeOn(Schedulers.computation())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(len -> {
            System.out.println(len);
        });
```

上面这段代码就是下面我们要分析的，但是一上来还是觉得这个复杂了，我再简单一点

```java
Observable.just("hello world!")
        .subscribe(System.out::println);
```

## just做了什么
我们现在一点点分析，首先我们知道平时我们创建一个Observable对象的时候是用利用下面的方式,非lambda表达式：

```java
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
		//do something...
    }
});
```
`just()`方法其实做了类似的事情，我们看下方法的具体实现：

```java
Observable.java
public static <T> Observable<T> just(final T value) {
    return ScalarSynchronousObservable.create(value);
}


ScalarSynchronousObservablejava
public static <T> ScalarSynchronousObservable<T> create(T t) {
    return new ScalarSynchronousObservable<T>(t);
}

protected ScalarSynchronousObservable(final T t) {
    super(RxJavaHooks.onCreate(new JustOnSubscribe<T>(t)));
    this.t = t;
}

static final class JustOnSubscribe<T> implements OnSubscribe<T> {
        final T value;

        JustOnSubscribe(T value) {
            this.value = value;
        }

        @Override
        public void call(Subscriber<? super T> s) {
            s.setProducer(createProducer(s, value));
        }
    }
```
这篇分析中我们一律跳过`RxJavaHooks`的内容，所以从上上面我们的比较我们知道，其实是创建了一个`JustOnSubscribe`对象，它是`OnSubscribe`实现类，这就是和我们平常用`create()`创建一个`Observable`对上了。

## 那么`OnSubscribe`的作用是什么呢？

```java
Observable.java
/**
 * Invoked when Observable.subscribe is called.
 * @param <T> the output value type
 */
public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {
    // cover for generics insanity
}
```
`subscribe()`被调用的时候就会调用`OnSubscribe`，也就是说，`OnSubscribe`是个回调，使它通知被订阅着自己被订阅了，可以开始发射数据了。这是怎么做到的呢？我们需要看看`subscribe()`的内部实现才能知道。

## subscribe做了什么？

`subscribe()`有多个重载方法，作用就是将传进来的`Action`对象包装成`Subscriber`对象，前面也提到了，数据流中只有三种事件，分别是包含某个值的事件，完成事件和错误事件，这里我们看一个最简单的重载方法，就是只传入了接收包含某个值事件的对象：


```java
Observable.java
public final Subscription subscribe(final Action1<? super T> onNext) {
    if (onNext == null) {
        throw new IllegalArgumentException("onNext can not be null");
    }
    
	//将另外接收另外两种事件的Action补上
    Action1<Throwable> onError = InternalObservableUtils.ERROR_NOT_IMPLEMENTED;
    Action0 onCompleted = Actions.empty();
    
    //包装成ActionSubscriber，它是Subscriber的实现类
    return subscribe(new ActionSubscriber<T>(onNext, onError, onCompleted));
}
```

下面才是`subscribe()`的真正实现：

```java
Observable.java
    public final Subscription subscribe(Subscriber<? super T> subscriber) {
        return Observable.subscribe(subscriber, this);
    }

    static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
     	 //省略参数检查代码
     	
        //通知subscriber和Observable联系起来了，也就是这里订阅发生了
        //方法内默认不做任何操作
        subscriber.onStart();
        
        // 如果不是传入的不是SafeSubscriber就包装成SafeSubscriber
        if (!(subscriber instanceof SafeSubscriber)) {
            subscriber = new SafeSubscriber<T>(subscriber);
        }

        //
        try {
            //调用onSubscribe的call方法，发射数据
            RxJavaHooks.onObservableStart(observable, observable.onSubscribe).call(subscriber);
            return RxJavaHooks.onObservableReturn(subscriber);
        } catch (Throwable e) {
            //错误处理代码
        }
    }
```

从上面可以看到，onSubscribe中传入的subscriber就是我们自己定义的订阅者，如果我们在`call()`方法中直接调用`subscriber.onNext(T t)`，被订阅者`Observable`中发生的数据订阅者就可以接收到了。

**这里需要注意一点，非常重要，就是`observable.onSubscribe.call(subscriber)`是在`subscribe()`中被调用的，所以`subscribe()`发生在什么线程，我们的`observable.onSubscribe.call(subscriber)`就发生在什么线程。其实`subscribeOn()`切换上游线程的原理，就是在某个线程中订阅了上游的被订阅者，从而影响线程的**

好了，既然现在我们的例子用的just，我们还是说明白just是怎么发射数据的。

## JustOnSubscribe
```java
ScalarSynchronousObservable.java

static final class JustOnSubscribe<T> implements OnSubscribe<T> {
    final T value;

    JustOnSubscribe(T value) {
        this.value = value;
    }

    @Override
    public void call(Subscriber<? super T> s) {
        s.setProducer(createProducer(s, value));
    }
}


```
