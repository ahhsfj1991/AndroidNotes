# #Dagger2.11学习笔记
## #目的
之前版本的dagger2依赖注入方式需要在activity中添加如下代码：

```java
public class MyActivity extends Activity {
  @Inject 
  Person person;

  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    DaggerActivityComponent
                .builder()
                .appComponent(((MyApplication)this.getApplication()).getAppComponent())
                .build()
                .inject(this);
    person.getName;            
  }
}
```
`DaggerActivityComponent`是dagger为我们生成的代码，依赖注入器，它是根据我们编写的这样的代码自动生成的：

```java
@Component
public interface ActivityComponent {
    void inject(MainActivity mainActivity)
}
```
为了减少代码，当然可以将第一段代码写到`BaseActivity`中，但是还是免不了需要在每个具体的activity中实现`inject(this)`。

那么有没有更加**无痛**的注入方式呢，下面介绍的dagger2.11应该可以做到

## #使用方法一
先建立一个需要注入的类以供使用

```kotlin
data class Person(var name: String, var age: Int){
    init {
        Log.e("dagger2demo", this.toString())
    }
}
```

然后是提供依赖的personmodule

```java
@Module
public class FangjunModule {
    @Provides
    public Person provideFangjun() {
        return new Person("zhoujielun", 25);
    }
}
```
然后是subcomponent接口

```java
@Subcomponent(modules = FangjunModule.class)
public interface MainActivitySubcomponent extends AndroidInjector<MainActivity> {
    @Subcomponent.Builder
    abstract class Builder extends AndroidInjector.Builder<MainActivity>{

    }
}
```
接着

```java
@Module(subcomponents = MainActivitySubcomponent.class)
public abstract class MainActivityModule {

    @Binds
    @IntoMap
    @ActivityKey(MainActivity.class)
    abstract AndroidInjector.Factory<? extends Activity>
    bindActivityInjectorFactory(MainActivitySubcomponent.Builder builder);
}
```
接着我们熟悉的component

```java
@Singleton
@Component(modules = {AndroidInjectionModule.class, MainActivityModule.class})
public interface AppComponent {
    void inject(MainApp mainApp);//MainApp是项目中自定义的application
}
```
make一下，就可以写application中的代码了

```java
public class MainApp extends Application implements HasActivityInjector{
    //这里的AndroidInject通过生成代码可以知道，其实就是dagger根据
    //MainActivitySubcomponent生成的 
    //DaggerAppComponent中的MainActivitySubcomponentImpl
    @Inject
    DispatchingAndroidInjector<Activity> mActivityInjector;

    private AppComponent mAppComponent;
    @Override
    public void onCreate() {
        super.onCreate();
        mAppComponent = DaggerAppComponent.builder().build();
        mAppComponent.inject(this);
    }

    @Override
    public AndroidInjector<Activity> activityInjector() {
        return mActivityInjector;
    }
}
```
怎么在activity中使用呢，更简单

```kotlin
class MainActivity : AppCompatActivity() {

    @Inject lateinit var fangjun: Person

    override fun onCreate(savedInstanceState: Bundle?) {
        AndroidInjection.inject(this)
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        txt_main_show.text = fangjun.name
    }
}
```
>其实要写这么多类，确实是变麻烦了，但是不要紧dagger为我们提供了ContributesAndroidInjector注解可以让我们不用写Subcomponent了

## #使用方式二
不写Subcomponent，那么应该怎么使用呢？只要在module中使用ContributesAndroidInjector注解就可以了：

```java
@Module
public abstract class MainFragmentModule {
    @ContributesAndroidInjector(modules = FangjunModule.class)
    abstract MainFragment contributeMainFragment();
}
```
ContributesAndroidInjector注解的作用和写Subcomponent效果一样，会在DaggerAppComponent中生成相应的代码。
>不过这个方式有个限制，那就是你需要写的Subcomponent中没有其他方法或者超类型才能使用

`@ContributesAndroidInjector(modules = FangjunModule.class)`就是需要注入的类

下面需要做两件事，1.装载MainFragmentModule到component中。2.修改一下MainActivity:

```java
@Singleton
@Component(modules = {AndroidInjectionModule.class,
        MainActivityModule.class,
        MainFragmentModule.class})
public interface AppComponent {
    void inject(MainApp mainApp);
}
```
```kotlin
//1.实现了HasSupportFragmentInjector接口
class MainActivity : AppCompatActivity(), HasSupportFragmentInjector {

    @Inject lateinit var fangjun: Person
    //2.需要注入DispatchingAndroidInjector
    //根据上面的讲解，其实就知道这里就是使用ContributesAndroidInjector注解生成的代码
    //在第一中方式中就是书写的subcomponent生成的代码
    @Inject lateinit var mFragmentInjector: DispatchingAndroidInjector<Fragment>

    override fun onCreate(savedInstanceState: Bundle?) {
        AndroidInjection.inject(this)
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        txt_main_show.text = fangjun.name
        supportFragmentManager.beginTransaction().add(fl_main_content.id, MainFragment()).commit()
    }

    override fun supportFragmentInjector(): AndroidInjector<android.support.v4.app.Fragment> = mFragmentInjector
}
```
最后需要在我们的fragment中注入依赖

```kotlin
class MainFragment : Fragment() {
    @Inject lateinit var fangjun: Person
    
    ...

    override fun onAttach(context: Context?) {
        super.onAttach(context)
        AndroidSupportInjection.inject(this)
    }
}
```

## #模块化中使用方法
现在安卓项目越来越大，模块化／组件化技术越来越重要，业务需要隔离。多模块中如何使用呢？还是接着上面的demo介绍。新建一个module，添加如下三个类（最简化状态）：

1.Module依赖提供类

```java
@Module
public class DaggerModule {
    private final Application mApplication;

    public DaggerModule(Application application) {
        mApplication = application;
    }

    @Singleton
    @Provides
    public Application provideApplication() {
        return this.mApplication;
    }
    
    //其他模块可能都需要的依赖，在这里提供
    @Singleton
    @Provides
    public Car provideCar() {
        return new Car("audi");
    }
}
```

2.componet

```java
@Singleton
@Component(modules = {AndroidInjectionModule.class, DaggerModule.class})
public interface DaggerComponent {
    Application application();
    Car getCar();
    void inject(DaggerDelegate daggerDelegate);
}
```

3.DaggerDelegate依赖注入管理类，方便管理，顶级依赖注入

```java
public class DaggerDelegate {

    private DaggerComponent mComponent;
    private final Application mApplication;

    public DaggerDelegate (Application application) {
        mApplication = application;
    }

    public void onCreate() {
        //顶级依赖注入
        mComponent = DaggerDaggerComponent.builder()
                .daggerModule(new DaggerModule(mApplication))
                .build();
        //mComponent.inject(this);
    }
    
    //为什么需要提供这个依赖，因为别的模块的component需要依赖这个顶级依赖
    public DaggerComponent getComponent() {
        return mComponent;
    }
}
```

**好了，现在就需要看看在别的模块中如何使用了**

第一步：修改AppComponent

```java
@AppScope
@Component(dependencies = DaggerComponent.class,//变化就是这里，我们需要依赖顶级依赖
        modules = {AndroidInjectionModule.class,
        MainActivityModule.class,
        MainFragmentModule.class})
public interface AppComponent {
    void inject(MainApp mainApp);
}
```

第二步：修改我们MainApp

```java
public class MainApp extends Application implements HasActivityInjector{
    ...
    
    @Override
    public void onCreate() {
        super.onCreate();

        DaggerDelegate delegate = new DaggerDelegate(this);
        delegate.onCreate();

        mAppComponent = DaggerAppComponent.builder()
                .daggerComponent(delegate.getComponent())
                .build();
        mAppComponent.inject(this);
    }

    ...
}
```

然后我们就可以在我们想要奥迪车的时候拿到车子啦

```kotlin
class MainActivity : AppCompatActivity(), HasSupportFragmentInjector {

    @Inject lateinit var fangjun: Person
    @Inject lateinit var mFragmentInjector: DispatchingAndroidInjector<Fragment>
    @Inject lateinit var mCar: Car

    override fun onCreate(savedInstanceState: Bundle?) {
        AndroidInjection.inject(this)
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        txt_main_show.text = fangjun.name + mCar.name
        supportFragmentManager.beginTransaction().add(fl_main_content.id, MainFragment()).commit()
    }

    override fun supportFragmentInjector(): AndroidInjector<android.support.v4.app.Fragment> = mFragmentInjector
}
```

## #原理分析
到这里，其实我们知道是怎么使用的了，但还是有两个疑问没有解开：

1. 顶级依赖怎么注入的
2. 非顶级依赖怎么注入的

现解答顶级依赖怎么注入的，非常简单，直接看生成的代码

```java
//DaggerAppComponent.java
this.getCarProvider = new com_example_common_DaggerComponent_getCar(builder.daggerComponent);
...

public static final class Builder {
    private DaggerComponent daggerComponent;
    
	//这个就是application代码中我们用到的
    public Builder daggerComponent(DaggerComponent daggerComponent) {
      this.daggerComponent = Preconditions.checkNotNull(daggerComponent);
      return this;
    }
  }
...

private final class MainActivitySubcomponentImpl implements MainActivitySubcomponent {
    ...

    @SuppressWarnings("unchecked")
    private void initialize(final MainActivitySubcomponentBuilder builder) {
    ....
    
      this.mainActivityMembersInjector =
          MainActivity_MembersInjector.create(
              provideFangjunProvider,
              dispatchingAndroidInjectorProvider,
              DaggerAppComponent.this.getCarProvider);
    }
  }
```
然后`daggerComponent.getCar()`的实现方式就是dagger的普通原理

下面我们关注一下，第二个问题，改变写法后，依赖是怎么注入的呢？
我们从`AndroidInjection.inject(this)`下手，看看内部代码怎么实现的：

```java
public final class AndroidInjection {
    private static final String TAG = "dagger.android";

    public static void inject(Activity activity) {
        Preconditions.checkNotNull(activity, "activity");
        Application application = activity.getApplication();
        if(!(application instanceof HasActivityInjector)) {
            throw new RuntimeException(String.format("%s does not implement %s", new Object[]{application.getClass().getCanonicalName(), HasActivityInjector.class.getCanonicalName()}));
        } else {
            AndroidInjector<Activity> activityInjector = ((HasActivityInjector)application).activityInjector();
            Preconditions.checkNotNull(activityInjector, "%s.activityInjector() returned null", application.getClass().getCanonicalName());
            activityInjector.inject(activity);
        }
    }

    public static void inject(Fragment fragment) {
        ...
    }

    private static HasFragmentInjector findHasFragmentInjector(Fragment fragment) {
        ...
    }

    public static void inject(Service service) {
        ...
    }

    public static void inject(BroadcastReceiver broadcastReceiver, Context context) {
        ...
    }

    public static void inject(ContentProvider contentProvider) {
        ...
    }

    private AndroidInjection() {
    }
}
```
这个类中提供了五种类型的注入，我们就看一下，这不就是获取application中继承的接口方法吗?然后调用DispatchingAndroidInjector实例的inject方法

```java
public final class DispatchingAndroidInjector<T> implements AndroidInjector<T> {
    private static final String NO_SUPERTYPES_BOUND_FORMAT = "No injector factory bound for Class<%s>";
    private static final String SUPERTYPES_BOUND_FORMAT = "No injector factory bound for Class<%1$s>. Injector factories were bound for supertypes of %1$s: %2$s. Did you mean to bind an injector factory for the subtype?";
    
    //存贮了所有需要注入的activity或者其他四种类型和其依赖提供工厂类
    private final Map<Class<? extends T>, Provider<Factory<? extends T>>> injectorFactories;

    @Inject
    DispatchingAndroidInjector(Map<Class<? extends T>, Provider<Factory<? extends T>>> injectorFactories) {
        this.injectorFactories = injectorFactories;
    }

    @CanIgnoreReturnValue
    public boolean maybeInject(T instance) {
        Provider<Factory<? extends T>> factoryProvider = (Provider)this.injectorFactories.get(instance.getClass());
        if(factoryProvider == null) {
            return false;
        } else {
            //通过实例对象获取其依赖提供工厂类
            Factory factory = (Factory)factoryProvider.get();

            try {
                //实例化
                AndroidInjector<T> injector = (AndroidInjector)Preconditions.checkNotNull(factory.create(instance), "%s.create(I) should not return null.", factory.getClass().getCanonicalName());
                
                //完成注入
                injector.inject(instance);
                return true;
            } catch (ClassCastException var5) {
                throw new DispatchingAndroidInjector.InvalidInjectorBindingException(String.format("%s does not implement AndroidInjector.Factory<%s>", new Object[]{factory.getClass().getCanonicalName(), instance.getClass().getCanonicalName()}), var5);
            }
        }
    }

    public void inject(T instance) {
        boolean wasInjected = this.maybeInject(instance);
        if(!wasInjected) {
            throw new IllegalArgumentException(this.errorMessageSuggestions(instance));
        }
    }

```
还要继续跟下代码才知道，依赖怎么注入的，那么AndroidInjector实例化都做了什么呢？看些接口定义：

```java
public interface AndroidInjector<T> {
    void inject(T var1);

    @DoNotMock
    public abstract static class Builder<T> implements AndroidInjector.Factory<T> {
        public Builder() {
        }

        public final AndroidInjector<T> create(T instance) {
            this.seedInstance(instance);
            return this.build();
        }

        @BindsInstance
        public abstract void seedInstance(T var1);

        public abstract AndroidInjector<T> build();
    }

    @DoNotMock
    public interface Factory<T> {
        AndroidInjector<T> create(T var1);
    }
}
```
还记得这些代码吗？

```java
@Subcomponent(modules = FangjunModule.class)
public interface MainActivitySubcomponent extends AndroidInjector<MainActivity> {
    @Subcomponent.Builder
    abstract class Builder extends AndroidInjector.Builder<MainActivity>{
    }
}
```
对，就是新写法需要添加的subcomponet，或者不自己手写，靠ContributesAndroidInjector注解自动生成。

*看看dagger根据这个代码为我们生成了什么*

```java
private final class MainActivitySubcomponentBuilder extends MainActivitySubcomponent.Builder {
    ...

    // 1
    @Override
    public MainActivitySubcomponent build() {
      ...
      // 2
      return new MainActivitySubcomponentImpl(this);
    }
    ...
  }

  private final class MainActivitySubcomponentImpl implements MainActivitySubcomponent {
    ...

    private MainActivitySubcomponentImpl(MainActivitySubcomponentBuilder builder) {
      assert builder != null;
      // 3
      initialize(builder);
    }

    @SuppressWarnings("unchecked")
    private void initialize(final MainActivitySubcomponentBuilder builder) {

      ...
      // 4
      this.mainActivityMembersInjector =
          MainActivity_MembersInjector.create(
              provideFangjunProvider,
              dispatchingAndroidInjectorProvider,
              DaggerAppComponent.this.getCarProvider);
    }

    @Override
    public void inject(MainActivity arg0) {
      mainActivityMembersInjector.injectMembers(arg0);
    }
  }
```
跟着上述步骤走，到第四步，我们应该看到熟悉的代码了，每个需要注入的activity，dagger都会为我们生成xxxActivity_MembersInjector类，他负责具体的我们需要的对象的注入。

最后我们看看DispatchingAndroidInjector<T>的实例化，在dagger2内部怎么做的：

```java
//DaggerAppComponent.java
      
      this.bindAndroidInjectorFactoryProvider =
          (Provider) DaggerAppComponent.this.mainFragmentSubcomponentBuilderProvider;

      this.bindAndroidInjectorFactoryProvider2 =
          (Provider) DaggerAppComponent.this.mainFragment2SubcomponentBuilderProvider;

      //注意下这里就可以了，看名字这些代码的意思就一目了然了
      this.mapOfClassOfAndProviderOfFactoryOfProvider =
          MapProviderFactory
              .<Class<? extends Fragment>, AndroidInjector.Factory<? extends Fragment>>builder(2)
              .put(MainFragment.class, bindAndroidInjectorFactoryProvider)
              .put(MainFragment2.class, bindAndroidInjectorFactoryProvider2)
              .build();

      this.dispatchingAndroidInjectorProvider =
          DispatchingAndroidInjector_Factory.create(mapOfClassOfAndProviderOfFactoryOfProvider);
```

项目demo地址[https://github.com/ahhsfj1991/Dagger2Demo](https://github.com/ahhsfj1991/Dagger2Demo)