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