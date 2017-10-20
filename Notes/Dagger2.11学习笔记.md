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
项目demo地址[https://github.com/ahhsfj1991/Dagger2Demo](https://github.com/ahhsfj1991/Dagger2Demo)