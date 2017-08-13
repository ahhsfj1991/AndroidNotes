# view的绘制
## view的诞生
### 加载xml布局文件
一切从`setContentView(@LayoutRes int layoutResID)`开始。

```java
public class Activity {

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
    }
}
```
acticity通过这个方法加载我们的布局文件，那么我们看看这个方法的源码：

```java
public void setContentView(@LayoutRes int layoutResID) {
    //getWindow()返回的是mWindow
    //而mWindow = new PhoneWindow(this, window);
    //PhoneWindow是Window的唯一实现类，是android系统中窗口的顶级类
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```
我们看下在`PhoneWindow`中`setContentView `的具体实现：

```java
@Override  
public void setContentView(int layoutResID) {  
	//在渲染布局资源前做一些前期准备工作
	//mContentParent是负责加载内容的容器
	//首先判断mContentParent是否为null
	if (mContentParent == null) {  
		installDecor();  
	} else {  
	  	//如果不为null，说明原来页面上已经有内容了，
	  	//所以我们要移除所有的内容，后面再加载新的内容上去
		mContentParent.removeAllViews();  
	}  
  //调用mLayoutInflater来根据我们的布局资源id渲染视图
mLayoutInflater.inflate(layoutResID, mContentParent);  
.....
}
```
从上面的代码可以知道如果`mContentParent `为`null`，就必须初始化`mContentParent `以便后续加载视图。`installDecor() `就是负责这项工作：

```java
private void installDecor() {
        //mDecor是window下的一个内部类DecorView，他是window用来填充视图的容器
        if (mDecor == null) {
        	//generateDecor创建mDecor
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            //利用mDecor创建mContentParent
            mContentParent = generateLayout(mDecor);
        }
    }
}
```

这里看幅图就知道DecorView的作用：
![Rendering preferences pane](http://img.blog.csdn.net/20160522211749849)

从上图我们可以知道，`DecorView`是整个控件树中的顶级view，我们调用`setContentView()`其实是实例化了`DecorView`,`DecorView`包含了两个子布局，一个是状态栏，一个是内容布局，现在知道为什么要叫这个方法名字了`setContentView()`，因为我们加载的就是内容布局。

###何时开始我们常说的测量、布局、绘制？
以上的分析只是介绍了加载布局文件的过程，并没有任何测量、布局、绘制的影子。其实这才是正常的，现在activity还处于不可见的状态，为什么要耗费资源去完成测绘流程呢？既然说到这，我们可以知道：

**什么时候activity开始可见了，也是view测绘流程开始的时候**


我们都知道`onResume()`方法执行后，activity处于可见状态，这个时候应该就是view开始测绘的时候，好现在我们到管理activity的`ActivityThread `类中找到调用activity的`onResume()`的源码：

```java
final void handleResumeActivity(IBinder token, boolean   clearHide, boolean isForward) { 
  ......
  //可以看到，这里执行了activity的onResume方法
  ActivityClientRecord r = performResumeActivity(token, clearHide); 
  if (r != null) {
    final Activity a = r.activity;
    .......
    if (r.window == null && !a.mFinished && willBeVisible) {

        //获得window对象
        r.window = r.activity.getWindow();

        //从window中获取DecorView对象
        View decor = r.window.getDecorView(); 
        decor.setVisibility(View.INVISIBLE);

        //从activity中获得与之关联的windowManager对象
        ViewManager wm = a.getWindowManager(); 
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        l.softInputMode |= forwardBit;
        if (a.mVisibleFromClient) {
            a.mWindowAdded = true;
            
            //这里将decor与WindowManager关联上，也就是将我们的decor正式添加到window
            wm.addView(decor, l); 
            
        }
        ......
    }
  }
}
```
### tips：

在这里提一句，在`wm.addView(decor, l)`中最后调用了`ViewRootImpl `的`setView()`方法，至此，`Window`、`WindowManager`、`ViewRootImpl `、`DecorView`四个主要的涉及到的类全部出现，这里只能简单的说一下三个的作用：


###`DecorView`
view树的顶级view

***

###`ViewRootImpl `
看下类说明:

>
 The top of a view hierarchy, implementing the needed protocol between View
 and the WindowManager.  This is for the most part an internal implementation
 detail of {@link WindowManagerGlobal}
>
>它是view结构层次中最顶层的，实现了View和WindowManager之间所需的协议，大部分都是WindowManagerGlobal的内部实现。

***

### `WindowManager`
WindowManager是一个抽象类,这个WindowManager的具体实现实在WindowManagerImpl中，看下类说明：

>The interface that apps use to talk to the window manager.

>app用于和window manager通信的接口。

其实就是和WindowManagerService(WMS)进行通信,也是WMS识别View具体属于那个Activity的关键。这里不展开说明，因为现在我也不知道！

***

### `Window`
在Android中,Window是个抽象的概念,Android中Window的具体实现类是PhoneWindow,Activity和Dialog中的Window对象都是PhoneWindow

***

用下面一幅图再次说明它们之间的关系：

![Rendering preferences pane](http://upload-images.jianshu.io/upload_images/1437930-f3d1fc6ba25292b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

好，现在接着刚才的`wm.addView(decor, l)`讲，这个方法我们需要到WindowManager的实现类WindowManagerImpl中找到这个方法的具体实现：

```java
 public void addView(View view, ViewGroup.LayoutParams params) {
  //这里通过mGlobal调用addView进行添加，而mGlobal是WindowManagerGlobal的一个内部实例
  //WindowManagerImpl其实是WindowManagerGlobal的代理类，另外还管理Dislay对象，WindowManagerGlobal才是真正管理所有的view
  mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

接着看WindowManagerGlobal.addView的实现：

```java
 public void addView(View view, ViewGroup.LayoutParams params,
      Display display, Window parentWindow) {
  ......
  //注意这个对象
  ViewRootImpl root;
  View panelParentView = null;

  synchronized (mLock) {
      ......
      //这里的view就是decor，利用DecorView和display实例化一个ViewRootImpl
      //display的类说明是:
      //Provides information about the size and density of a logical display
      //由此可以知道他是提供一些逻辑显示的大小和密度的信息
      root = new ViewRootImpl(view.getContext(), display); 
      view.setLayoutParams(wparams);
      mViews.add(view);
      mRoots.add(root);
      mParams.add(wparams);
  }
  try {
      //这里调用了ViewRootImpl的setView方法，将DecorView与ViewRootImpl产生来关联。
      root.setView(view, wparams, panelParentView); 
  } catch (RuntimeException e) {
      synchronized (mLock) {
         final int index = findViewLocked(view, false);
         if (index >= 0) {
           removeViewLocked(index, true);
         }
      }
      throw e;
  }
}
```

接着看：

```java
public final class ViewRootImpl implements ViewParent,  
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {  
  synchronized (this) {  
      if (mView == null) {  
         mView = view;  
         ......  
         if (view instanceof RootViewSurfaceTaker) {  
           mSurfaceHolderCallback = 
          ((RootViewSurfaceTaker)view).willYouTakeTheSurface();  
           if (mSurfaceHolderCallback != null) {  
            mSurfaceHolder = new TakenSurfaceHolder();  
            mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);  
           }  
         }  

       //这是我们很熟悉的一个方法,这就是绘制我们的view的入口
       requestLayout();
         ......  

      }  
  }  
}  

......  
}
```
而`requestLayout() ----> scheduleTraversals() -----> performTraversals ()`


*performTraversals()*也算是在view的测绘中很重要的一个方法，它的具体实现是：

```java
private void performTraversals() {
      ...
  if (!mStopped) {
      //获取顶层布局的childWidthMeasureSpec
      int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);  
      //获取顶层布局的childHeightMeasureSpec
      int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
      //测量开始测量
      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);       
      }
  } 

  if (didLayout) {
      //执行布局方法
      performLayout(lp, desiredWindowWidth, desiredWindowHeight);
      ...
  }
  if (!cancelDraw && !newSurface) {
   ...
         //开始绘制了哦
         performDraw();
      }
  } 
```


至此view的view测绘前的种种准备总算讲完了，这里加一个小知识点:
>* invalidate()

>>请求重绘 View 树，即 draw 过程，假如视图发生大小没有变化就不会调用layout()过程，并且只绘制那些调用了invalidate()方法的 View。

>* requestLayout()

>>子View调用requestLayout方法，会标记当前View及父容器，同时逐层向上提交，直到ViewRootImpl处理该事件，ViewRootImpl会调用三大流程，从measure开始，对于每一个含有标记位的view及其子View都会进行测量、布局、绘制。

## 三大流程
整个 View 树的绘图流程在ViewRoot.java类的performTraversals()函数展开，该函数所做 的工作可简单概况为是否需要重新计算视图大小(measure)、是否需要重新安置视图的位置(layout)、以及是否需要重绘(draw)，流程图如下：
![Rendering preferences pane](https://github.com/android-cn/android-open-project-analysis/raw/master/tech/viewdrawflow/image/view_mechanism_flow.png)

## Measure
### 理解MeasureSpec
首先让我们看下官方文档：

>A MeasureSpec encapsulates the layout requirements passed from parent to child. Each MeasureSpec represents a requirement for either the width or the height. A MeasureSpec is comprised of a size and a mode. There are three possible modes:

>MeasureSpec对象中封装了从父对象传递给孩子的布局所需数数据（你要成为我的子控件，你要在我里面占位置，你先要知道我有多少空间吧？）。每一个MeasureSpec对象包含了对于宽度和高度的描述（也就是父控件告诉子控件，我有多大点地和我对于空间的使用策略等）。 MeasureSpec由大小和模式组成。有三种可能的模式：

>* UNSPECIFIED 父控件还不知道子控件的大小，对子控件也没有任何约束，说你想占多少地方就占吧。（这个一般很少用到）
* EXACTLY 这种状态下的控件的大小是明确的。
* AT_MOST 父控件对子控件说，我还不知道你的大小，我给你自由，我的地方是这么大，你按你的意愿来，但最大也只能跟我一样大了

MeasureSpec代表一个32位的int值，高2位代表SpecMode，低30位代表SpecSize，它就好比一个说明书，里面写好了子空间能够操控的空间和子空间能够建造的模式，我们看下一个生成一个MeasureSpec的方法:

```java
private static final int MODE_SHIFT = 30;
private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

public static int makeMeasureSpec(int size,int mode) {
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
        
public static int getMode(int measureSpec) {
    return (measureSpec & MODE_MASK);
}
        
public static int getSize(int measureSpec) {
    return (measureSpec & ~MODE_MASK);
}
```
还是从方法`performTraversals()`开始，`int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width)`获取顶层布局的childWidthMeasureSpec,那我们看下`getRootMeasureSpec `的实现：

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {

    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        //mView就是DecorView,顶级view终于拿到了自己的说明书
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```
前面说了DecorView是一个FrameLayout，那我们就找它的measure方法实现，结果发现没有，我们找它的父类ViewGroup，还是没有，继续找到ViewGroup的父类View，发现了measure方法,这里我们只留关键代码：

```java
public final void measure(int widthMeasureSpec,int  heightMeasureSpec) {
  ...

  onMeasure(widthMeasureSpec, heightMeasureSpec);
  ...
}
```

### FrameLayout(ViewGroup)的onMeasure方法
现在我们回到FrameLayout中看看它是怎么实现onMeasure方法的：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
  //获得frameLayout下childView的个数
  int count = getChildCount();
  
  //看这里的代码我们可以根据前面的Measure图来进行分析，因为只要parent
  //不是EXACTLY模式，以frameLayout为例，假设frameLayout本身还不是EXACTL模式，
  //那么表示他的大小此时还是不确定的，从表得知，此时frameLayout的大小是根据
  //childView的最大值来设置的，这样就很好理解了，也就是childView测量好后还要再
  //测量一次，因为此时frameLayout的值已经可以算出来了，对于child为MATCH_PARENT
  //的，child的大小也就确定了，理解了这里，后面的代码就很 容易看懂了
  final boolean measureMatchParentChildren =
          MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
          MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
  
  //清理存储模式为MATCH_PARENT的child的队列
  mMatchParentChildren.clear();
  
  //下面三个值最终会用来设置frameLayout的大小
  int maxHeight = 0;
  int maxWidth = 0;
  int childState = 0;
  
  //开始便利frameLayout下的所有child
  for (int i = 0; i < count; i++) {
      final View child = getChildAt(i);
      
      //只要mMeasureAllChildren是true，就算child是GONE也会被测量哦，
      if (mMeasureAllChildren || child.getVisibility() != GONE) {
          
          /*********************开始测量childView ************************/
          measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);

          //下面代码是获取child中的width 和height的最大值，后面用来重新设置frameLayout，有需要的话
          final LayoutParams lp = (LayoutParams) child.getLayoutParams();
          maxWidth = Math.max(maxWidth,
                  child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
          maxHeight = Math.max(maxHeight,
                  child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
          childState = combineMeasuredStates(childState, child.getMeasuredState());

          //如果frameLayout不是EXACTLY
          if (measureMatchParentChildren) {
              //注意这里用的是或，所以下面在处理这些child的时候需要分情况处理
              if (lp.width == LayoutParams.MATCH_PARENT ||
                      lp.height == LayoutParams.MATCH_PARENT) {
                  
                  //存储LayoutParams.MATCH_PARENT的child，因为现在还不知道frameLayout大小，
                  //也就无法设置child的大小，该集合中存储的child后面需重新测量
                  mMatchParentChildren.add(child);
              }
          }
      }
  }
  //循环结束，遍历所有child后终于找到最大的大小了
    ....
  //这里开始设置frameLayout的大小
  setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),resolveSizeAndState(maxHeight, heightMeasureSpec,childState << MEASURED_HEIGHT_STATE_SHIFT));

  //frameLayout大小确认了，我们就需要对宽或高为LayoutParams.MATCH_PARENTchild重新测量，设置大小
  count = mMatchParentChildren.size();
  if (count > 1) {
      for (int i = 0; i < count; i++) {
          final View child = mMatchParentChildren.get(i);
          final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
          final int childWidthMeasureSpec;
          //这里存在两种情况
          if (lp.width == LayoutParams.MATCH_PARENT) {
              final int width = Math.max(0, getMeasuredWidth()
                      - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                      - lp.leftMargin - lp.rightMargin);

              //如xml中是android:layout_width="match_parent"
              childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                      width, MeasureSpec.EXACTLY);
          } else {

              //如xml中是android:layout_width="100dp"
              childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                      getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                      lp.leftMargin + lp.rightMargin,
                      lp.width);
          }

          //这里是对高做处理，与宽类似
          final int childHeightMeasureSpec;
          if (lp.height == LayoutParams.MATCH_PARENT) {
              final int height = Math.max(0, getMeasuredHeight()
                      - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                      - lp.topMargin - lp.bottomMargin);
              childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                      height, MeasureSpec.EXACTLY);
          } else {
              childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                      getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                      lp.topMargin + lp.bottomMargin,
                      lp.height);
          }

          /****************最终，再次测量child***********************/
          child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
      }
  }
}
```
我们再看下第一次测量child的方法，为什么说它进行了child的测量呢，一看源码便知：

```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    //child得到自己的说明书，从参数就可以知道child的说明书需要参考parent的说明书和它自己的一些LayoutParams，此外还有view的Margin和Padding
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);
    //拿到说明书了，当然可以开始测量了
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

### View的onMeasure方法
OK，现在一路走下来，从还差最后一点没说了，就是view树的最底层，也就是View的onMeasure方法：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```
1、调用setMeasuredDimension设置view的大小

2、调用getDefaultSize获取View的大小

3、getSuggestedMinimumWidth获取一个建议最小值

现在我们从最里面的方法开始看

```java
protected int getSuggestedMinimumWidth() {
	return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
  }
```
伪代码就是：
>
>（是否有背景）？android:minWidth=""设置的值 ：max（android:minWidth=""设置的值， bitmap形式下的实际宽度值）
>


```java
public static int getDefaultSize(int size, int measureSpec) {
	int result = size;
	//获得MeasureSpec的mode
	int specMode = MeasureSpec.getMode(measureSpec);
	//获得MeasureSpec的specSize
	int specSize = MeasureSpec.getSize(measureSpec);
		
	switch (specMode) {
		case MeasureSpec.UNSPECIFIED:
		result = size;
		break;
		case MeasureSpec.AT_MOST:
		case MeasureSpec.EXACTLY:
		//可以看到，最终返回的size就是我们MeasureSpec中测量得到的size
		result = specSize;
		break;
		}
	return result;
}
```
上面代码需要注意的就是AT_MOST与EXACTLY模式下是一样的，这在我们自定view的时候一定要处理

接着我们看setMeasuredDimension

```java
protected final void setMeasuredDimension(int   measuredWidth, int measuredHeight) {
  //判断是否使用视觉边界布局
  boolean optical = isLayoutModeOptical(this);
  //判断view和parentView使用的视觉边界布局是否一致
  if (optical != isLayoutModeOptical(mParent)) {
      //不一致时要做一些边界的处理
      Insets insets = getOpticalInsets();
      int opticalWidth  = insets.left + insets.right;
      int opticalHeight = insets.top  + insets.bottom;

      measuredWidth  += optical ? opticalWidth  : -opticalWidth;
      measuredHeight += optical ? opticalHeight : -opticalHeight;
  }
  //经过过滤之后调用了setMeasuredDimensionRaw方法，其实就是这个方法设置了view的大小
  setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
  //最终将测量好的大小存储到mMeasuredWidth和mMeasuredHeight上，所以在测量之后我们可以通过调用getMeasuredWidth获得测量的宽、getMeasuredHeight获得高
  mMeasuredWidth = measuredWidth;
  mMeasuredHeight = measuredHeight;

  mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}

```
### tips
对于以上一个小知识点视觉边界布局的一个小补充，其实视觉边界是相对控件边界布局说的，下面两个图是显示了每个控件边界的图（蓝色 为控件的边界；粉红色为视觉边界）：

*控件边界布局*

![Rendering preferences pane](http://yunzaiqianfeng.b0.upaiyun.com/android/clickbound.jpg)

*视觉边界布局*

![Rendering preferences pane](http://yunzaiqianfeng.b0.upaiyun.com/android/opticalbound.jpg)

## Layout

从入口`performLayout`开始

```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
int desiredWindowHeight) {
    mLayoutRequested = false;
    mScrollMayChange = true;
    mInLayout = true;
	//注意这里，上面提及过，mView就是DecorView
    final View host = mView;
    if (DEBUG_ORIENTATION || DEBUG_LAYOUT) {
         Log.v(TAG, "Laying out " + host + " to (" +
            host.getMeasuredWidth() + ", " + host.getMeasuredHeight() + ")");
    }

    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
    try {
      //调用了host.layout
      host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight()); 
      mInLayout = false;

    .....
    } finally {
       Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    mInLayout = false;
}
```
其实想想也知道，肯定是从顶级view`DecorView `开始布局，我们注意下刚刚传进来的参数`host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());`这四个参数按顺利所代码的含义分别是left，top，right，bottom，也就是左、上、右、下.既然是顶级view，当然应该贴着整个屏幕，所以left和top都是0

所有view都是继承自`View`我们看下它的layout方法:

```java
public void layout(int l, int t, int r, int b) {
  //还记得上面的tips吗？这里就是一些flag的判断，比如这里，layout前没有measure的话，需要先进行measure操作
  if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
      onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
      mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
      }
      ......
      
      //isLayoutModeOptical(mParent)判断是传统模式还是视觉模式，上面已经提及
      //然后对不同模式分别调用对象的方法，作用是设置View的四个点
      boolean changed = isLayoutModeOptical(mParent) ?
         setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

      if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
      //直接调用onLayout方法进行布局
      onLayout(changed, l, t, r, b);
      ......
       for (int i = 0; i < numListeners; ++i) {
         //如果设置了OnLayoutChangeListener，在layout之后就会回调告诉你
         listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
       }
    
    }
}
```
可以注意到`onLayout(changed, l, t, r, b)`还传了一个changed参数，其实它表示新传进来的四个顶点参数和之前的有没有变化，可以看下它的实现：

```java
protected boolean setFrame(int left, int top, int right, int bottom) {
     boolean changed = false;
     ......
     //判断宽高是否有分生变化
     boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

     //Invalidate our old position
     //如果大小变化了，在已绘制了的情况下就请求重新绘制
     invalidate(sizeChanged);

     //将新的值存储到view的成员变量中
     mLeft = left;
     mTop = top;
     mRight = right;
     mBottom = bottom;
     ......
    return changed;
}
```

说到这里我们应该看下布局真正的实现方法`onLayout`了，这里需要分两种情况：

* View的onLayout
* ViewGroup的onLayout

我们先看下View的onLayout的方法实现：

```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
  }
```
既然是空的，其实也很容易理解，你在什么位置，你父view早就按照自己的规范，把你安排好了，你也没有子view需要安排

*so：* 我们看看有子view可以安排的ViewGroup的onLayout的方法，因为这是个抽象方法，我们还是挑FrameLayout看它的具体实现：

```java
	@Override
	protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
	
	    layoutChildren(left, top, right, bottom, false /* no force left gravity */);
	}

    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        //子view的数量
        final int count = getChildCount();
		  //父view左面位置，getPaddingLeftWithForeground获得的是对应的内边距
        final int parentLeft = getPaddingLeftWithForeground();
        //父view右边面位置
        final int parentRight = right - left - getPaddingRightWithForeground();

        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();
		  //遍历所有子view，进行布局
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();

                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();

                int childLeft;
                int childTop;

                ......

				//针对不同的水平方向Gravity做处理
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        ......
                    case Gravity.RIGHT:
                        ......
                    case Gravity.LEFT:
                    default:
                        childLeft = parentLeft + lp.leftMargin;
                }
                //针对不同的垂直方向Gravity做处理
                switch (verticalGravity) {
                    case Gravity.TOP:
                        ......
                    case Gravity.CENTER_VERTICAL:
                        ......
                    case Gravity.BOTTOM:
                        ......
                    default:
                        childTop = parentTop + lp.topMargin;
                }
				//用过帧布局的应该知道为什么刚才上面为什么只计算childLeft, childTop了
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
```

## Draw

首先让我们看一幅时序图
![Rendering preferences pane](http://upload-images.jianshu.io/upload_images/1811893-ed8f9fd302253f8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>从图中可以看到从入口`performDraw()`开始，并没有直接调用顶级view`DecorView`的draw方法，中间经过了几个方法，这里我们不具体看实现了，我们知道无论是测量布局还是现在讨论的绘制，都是measure（）、layout（）或者现在的draw（）在调用它们各自的onXXX()方法，然后调用子view的XXX()方法。这样不断的递归下去。

现在我们暂时不看View的draw（）方法，我们先看时序图中的8，9，10。而8方法dispatchDraw在View和ViewGroup有不同的实现

* View

```java
//当然应该是空的，哪来的子view可以通知啊
protected void dispatchDraw(Canvas canvas) {

}
```

* ViewGroup

```java
@Override
protected void dispatchDraw(Canvas canvas) {
    boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);

    //获取child的数量
    final int childrenCount = mChildrenCount;
    final View[] children = mChildren;

    ......
    for (int i = 0; i < childrenCount; i++) {
       ......

       //调用drawChild传递canvas、child进去绘制child
       more |= drawChild(canvas, child, drawingTime);
       
    }
    ......
}

boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {

    ......
    if (!drawingWithDrawingCache) {
       
       // 这里调用子View的draw方法，并将调整好的canvas传进去
       draw(canvas);
         
    } else if (cache != null) 
       // 如果是cache模式，则利用cache代码不贴了
       ......
    }
    ......

}
```

>*为什么不是直接调用draw(Canvas canvas) ？*

>答：因为从性能的考虑，应该优先考虑缓存中拿对象绘制，不行我们再调用draw(Canvas canvas)全部重新画一遍


好现在看我们的大boss`draw(Canvas canvas)`

```java
**
     * Manually render this view (and all of its children) to the given Canvas.
     * The view must have already done a full layout before this function is
     * called.  When implementing a view, implement
     * {@link #onDraw(android.graphics.Canvas)} instead of overriding this method.
     * If you do need to override this method, call the superclass version.
     *
     * @param canvas The Canvas to which the View is rendered.  
     *
     * 根据给定的 Canvas 自动渲染 View（包括其所有子 View）。在调用该方法之前必须要完成 layout。当你自定义 view 的时候，
     * 应该去是实现 onDraw(Canvas) 方法，而不是 draw(canvas) 方法。如果你确实需要复写该方法，请记得先调用父类的方法。
     */
    public void draw(Canvas canvas) {
    
        / * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background if need
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children (dispatchDraw)
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

 	// Step 1, draw the background, if needed
        if (!dirtyOpaque) {
            drawBackground(canvas);
        }
        
         // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            // Step 6, draw decorations (scrollbars)
            onDrawScrollBars(canvas);

            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // we're done...
            return;
        }
        
        // Step 2, save the canvas' layers
        ...
        
        // Step 3, draw the content
        if (!dirtyOpaque) 
        	onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        
        // Step 6, draw decorations (scrollbars)
        onDrawScrollBars(canvas);
    }
```

# MVP vs MVVM

## MVP
![Rendering preferences pane](https://github.com/googlesamples/android-architecture/wiki/images/mvp.png)

> 优点: MVP架构可读性好、可测试性好
> 
> 缺点：相比MVVM需要维护更多的接口和代码

## MVVM
![Rendering preferences pane](https://github.com/googlesamples/android-architecture/wiki/images/mvvm-databinding.png)

> 优点: 相比MVP少一点类和代码，MVVM架构将UI逻辑从业务逻辑中分离出来
> 
> 缺点：找了很多地方，都没怎么提及，这里说一个我不太喜欢的理由，虽然DataBinding很强大，但是还有有些地方没办法使用，比如列表的展现，刷新，列表监听的添加，这个时候就需要类似MVP那样的实现方式，但是比如很多button的监听又是写在viewmodel中，但是我个人不喜欢在一个app中两种操作view方式混着用

上面是谷歌官方出的图，具体实现的代码就不贴了，相信大家都熟悉不过了，这里直接对比一下，可以看出，mvvm利用DataBinding库代替了mvp中x层得到数据后的回馈。

最后我在realm官方网站上找到的一篇文章[MVC vs. MVP vs. MVVM on Android](http://macdown.uranusjr.com/features/)中作者（realm的工程师）也说明了一个观点：

*So which pattern is best for you? If you’re choosing between MVP and MVVM, a lot of the decision comes down to personal preference, but seeing them in action will help you understand the benefits and tradeoffs*

选哪个？看自己。

