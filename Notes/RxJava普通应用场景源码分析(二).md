# RxJava普通应用场景源码分析(二)
## map和filter
这次我们要分析的就是在[RxJava普通应用场景源码分析(一)](https://github.com/ahhsfj1991/AndroidNotes/blob/master/Notes/RxJava%E6%99%AE%E9%80%9A%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(%E4%B8%80).md)中最简单的rxjava的形式上增加map和filter操作符。让我们看下下面这串代码：

```java
Observable.just("hello world!")
        .map(String::length)
        .filter(integer -> integer > 0)
        .subscribe(System.out::println);
```
上面这段代码的作用我想我不用在这里介绍了，我想先从`map`介绍，介绍完`map`再看filter的源码就非常简单了，或者说这一类的操作符其实都是差不多的实现。

## map(Func1<? super T, ? extends R> func)

```java
Observable.java
/**
 * @param func a function to apply to each item emitted by the Observable
 * func就是被用来处理每个被Observable发射出来额数据的函数
 * @return an Observable that emits the items from the source Observable, transformed by the specified function
 * 返回一个能够接收所有从初始Observable发出的数据，并利用函数进行转换的新的Observable
 */
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    //创建了一个OnSubscribe对象
    return unsafeCreate(new OnSubscribeMap<T, R>(this, func));
}
```

我们再看看这个OnSubscribe对象做了些什么：

```java
OnSubscribeMap.java
public OnSubscribeMap(Observable<T> source, Func1<? super T, ? extends R> transformer) {
    this.source = source;
    this.transformer = transformer;
}

@Override
public void call(final Subscriber<? super R> o) {
    //利用传入的func函数和下游订阅者(o)构造一个新的订阅者(MapSubscriber)
    MapSubscriber<T, R> parent = new MapSubscriber<T, R>(o, transformer);
    //便于统一取消订阅
    o.add(parent);
    //利用自己创建的MapSubscriber订阅source Observable
    source.unsafeSubscribe(parent);
}

```
我们先不看MapSubscriber具体做了什么，先理一下上面的逻辑。其实最下游的订阅方法订阅的并不是source Observable，而是`map`返回的Observable，在[RxJava普通应用场景源码分析(一)](https://github.com/ahhsfj1991/AndroidNotes/blob/master/Notes/RxJava%E6%99%AE%E9%80%9A%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(%E4%B8%80).md)中我们已经分析过了，订阅方法`subscribe()`中会调用这个Observable中的OnSubscribe#call()方法。call()实现如上代码。那么`map`的作用显而易见了，就是：

*我自己来订阅上游的Observable，拿到数据并转换之后再发送给你*

那么它是怎么拿到数据并转换之后再发送给下游的呢？答案就在：

## MapSubscriber

```java
OnSubscribeMap.java
static final class MapSubscriber<T, R> extends Subscriber<T> {

    final Subscriber<? super R> actual;

    final Func1<? super T, ? extends R> mapper;

    boolean done;

    public MapSubscriber(Subscriber<? super R> actual, Func1<? super T, ? extends R> mapper) {
        this.actual = actual;
        this.mapper = mapper;
    }

    @Override
    public void onNext(T t) {
        R result;

        try {
            //我们定义的func函数最终在这里将上游发送过来的数据进行转换
            result = mapper.call(t);
        } catch (Throwable ex) {
            //...
        }
        //发送给下游的订阅者
        actual.onNext(result);
    }

    //省略onError，onCompleted，setProducer
}
```

`map`到这里就分析完毕了，是不是很简单，前面提到了`filter`其实和`map`的实现是差不多的,所以`filter`也一定需要返回一个新的`Observable`并构建自己的`OnSubscribe`：

```java
public final Observable<T> filter(Func1<? super T, Boolean> predicate) {
    return unsafeCreate(new OnSubscribeFilter<T>(this, predicate));
}
```
它也一定需要构建一个自己的`Subscriber`并订阅上游`Observable`：

```java
OnSubscribeFilter.java
public void call(final Subscriber<? super T> child) {
    FilterSubscriber<T> parent = new FilterSubscriber<T>(child, predicate);
    child.add(parent);
    source.unsafeSubscribe(parent);
}
```
并在自己构造的`FilterSubscriber`中过滤数据，并发送给下游`Subscriber`:

```java
OnSubscribeFilter.java
public FilterSubscriber(Subscriber<? super T> actual, Func1<? super T, Boolean> predicate) {
    this.actual = actual;
    this.predicate = predicate;
    request(0);
}

public void onNext(T t) {
    boolean result;

    try {
        result = predicate.call(t);
    } catch (Throwable ex) {
        //...
    }

    if (result) {
        actual.onNext(t);
    } else {
        //这里和backpressure的知识有关，这里暂不提及
        request(1);
    }
}
```

