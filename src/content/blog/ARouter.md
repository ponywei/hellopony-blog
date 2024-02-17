---
pubDatetime: 2018-03-27
title: ARouter 的使用及基本原理
featured: false
draft: false
tags:
  - Android
description: ARouter是阿里巴巴开源的一个Android平台上的路由组件，为模块化之后的工程提供跨模块通信功能。在引入项目中使用的同时，也阅读了其源码进行了解和学习。
---

ARouter是阿里巴巴开源的一个Android平台上的路由组件，为模块化之后的工程提供跨模块通信功能。在引入项目中使用的同时，也阅读了其源码进行了解和学习。

<!-- more -->

# ARouter

Android平台中对页面、服务提供路由功能的中间件。[「GitHub ARouter」](https://github.com/alibaba/ARouter)

## 使用

### gradle配置

每个使用到此路由的Module都需要在build.gradle中添加如下编译和依赖配置（发起模块与目标模块都）：

```groovy
android {
    defaultConfig {
	...
	javaCompileOptions {
	    annotationProcessorOptions {
		arguments = [ moduleName : project.getName() ]
	    }
	}
    }
}

dependencies {
    // 替换成最新版本, 需要注意的是api要与compiler匹配使用，均使用最新版可以保证兼容
    compile 'com.alibaba:arouter-api:x.x.x'
    annotationProcessor 'com.alibaba:arouter-compiler:x.x.x'
    ...
}
```

### 混淆配置

如果项目使用混淆，需要keep一下：

```
-keep public class com.alibaba.android.arouter.routes.**{*;}
-keep class * implements com.alibaba.android.arouter.facade.template.ISyringe{*;}
```

### 初始化

初始化工作只需要在Application onCreate的时候执行一次即可，无需重复添加：

```java
//初始化ARouter
if (BuildConfig.DEBUG) {
    ARouter.openLog();
    ARouter.openDebug();
}
ARouter.init(getApplication());
```

### RouterPath

**注：ARouter使用Path对目标组件进行唯一标识，Path至少需要有两级，/xx/xxxx，一级路径名xx会被ARouter识别为Group名**；
为避免多模块使用导致路径管理混乱，在CommonBuiness中引入RouterPath类对各个Module的路由路径定义进行管理，如果租房模块RentModule中需要添加一个租房单页的跨模块路由页面，需要先在RouterPath中添加路径，代码如下：

```java
private static final String GROUP_RENT = "/rent/";

public static class Rent {
	public static final String DETAIL = GROUP_RENT + "detail";
}
```

### RouterService

为规范跨模块跳转的使用，引入RouterService对“**可能重复使用的**”路由跳转方法进行统一。如果某些页面仅有一个入口，可以忽略此步骤。如添加一个租房房源单页的带参数入口，在RouterService中添加如下路由方法：

```java
public static void startHouseDetailActivity(String propertyJson, String bp) {
        ARouter.getInstance().build(RouterPath.Rent.DETAIL)
                .withString(AnjukeConstants.Rent.EXTRA_FROM, bp)
                .withString(AnjukeConstants.Rent.EXTRA_PROPERTY_INFO, propertyJson)
                .navigation();
    }
```

### 添加注解

在对应目标路由页面的Activity添加@Route注解：

```java
@Route(path = RouterPath.Rent.DETAIL)
public class HouseDetailActivity extends AbstractBaseActivity{
...
}
```

### 参数携带

对于一般基本类型的参数携带，ARouter均可直接支持，也可使用`.withParcelable(key, object)`来传递对象，使用`withParcelableArrayList(key, objectArrayList)`来传递对象数组列表。

### 指定Intent Flag

某些场景下我们在跳转页面时需要在Intent中携带Flag信息`intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);`，在使用ARouter中的实现方法是在路由方法中`.withFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP)`。

```java
ARouter.getInstance().build("/home/main")
	.withFlags(ntent.FLAG_ACTIVITY_CLEAR_TOP);
	.navigation();
```

### startActivityForResult

`.navigation()`实现的是startActivity，`.navigation(context, requestCode)`实现的就是startActivityForResult；

```java
// navigation的第一个参数必须是Activity，第二个参数则是RequestCode
ARouter.getInstance().build("/home/main", "ap").navigation(this, 5);
```

### 获取Fragment

简单Fragment可以直接获取到，然后add到Activity即可。复杂Fragment如二手房列表PropListFragment和租房列表HouseListFragment，有内部接口需要实现加载回调等其他交互逻辑的，需要自行抽取基类和接口处理（具体参考代码）。

```java
Fragment fragment = (Fragment) ARouter.getInstance().build("/test/fragment").navigation();
```

### 拦截器

ARouter拦截器的经典应用，就是在跳转中处理登录事件，或其他需要暂停页面跳转等待用于确认操作的场景。
下面的代码是一个拦截器，在拦截到path为"/app/third"的跳转时，弹出一个确认继续的弹窗，在弹窗的点击事件中决定continue还是interrupt。
在安居客的登录应用场景中，登录操作需要打开1-2个activity，用户操作时长无法确认，而且登录的成功与否也无法简单的通过回调事件来通知（实际是使用广播），因此不建议在登录场景使用拦截器。

注：

1. 由于Interceptor执行在子线程中，AlertDialog的show方法需要手动切换到UI线程上来调用；
2. AlertDialog.Builder(context)需要context为Activity，而IInterceptor的init(context)中context为Application Context，所以还需要指定具体的Activity！

```java
@Interceptor(priority = 1,name = "TheThirdInterceptor")
public class ThirdInterceptor implements IInterceptor {
	private static final String TAG = ThirdInterceptor.class.getSimpleName();
	// This is an application context
	Context context;

	@Override
	public void process(final Postcard postcard, final InterceptorCallback callback) {
		if ("/app/third".equals(postcard.getPath())) {
			final AlertDialog.Builder builder = new AlertDialog.Builder(LibMainActivity.getActivity())
					.setCancelable(false)
					.setTitle("温馨提示")
					.setMessage("由于你触发了" + postcard.getPath() + "条件激活了拦截器，需要确认以下操作，是否继续？")
					.setPositiveButton("登录并继续", new DialogInterface.OnClickListener() {
						@Override
						public void onClick(DialogInterface dialog, int which) {
							postcard.withString("login_user", "pony");
							callback.onContinue(postcard);
						}
					})
					.setNeutralButton("继续", new DialogInterface.OnClickListener() {
						@Override
						public void onClick(DialogInterface dialog, int which) {
							callback.onContinue(postcard);
						}
					})
					.setNegativeButton("取消", new DialogInterface.OnClickListener() {
						@Override
						public void onClick(DialogInterface dialog, int which) {
							callback.onInterrupt(new Throwable("Manual cancel."));
						}
					});

			MainLooper.runOnUiThread(new Runnable() {
				@Override
				public void run() {
					builder.create().show();
				}
			});
		} else {
			callback.onContinue(postcard);
		}
	}

	@Override
	public void init(Context context) {
		this.context = context;
		Log.d(TAG, "init.");
	}
}****
```

### 注解参数

默认情况下，在路由方法中携带的参数是以Intent Extras的形式传递到目标页面的，旧的getIntentExtras()解析参数的方法仍然可以使用。以下要说的是ARouter提供的通过注解的方式自动解析URL中参数的方法，可做为新建目标页面的可选方案使用。（**注意：在使用@Autowired注解入参的情况下，需要在onCreate中添加注入代码`ARouter.getInstance().inject(this)`；被注解参数不能为private。**）。

```java
@Route(path = "/app/second")
public class SecondTestActivity extends AppCompatActivity {
	@Autowired
	protected String name;
	@Autowired
	public int age;
	@Autowired
	boolean isGraduated;

	@Autowired
	User friend;

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_second_test);

		ARouter.getInstance().inject(this);
		((TextView) findViewById(R.id.textView)).setText(String.format("Name:%s\nAge:%d\nGraduated:%b\nFriend:%s",
				name, age, isGraduated, friend.toString()));
	}
}
```

## 基本原理

对源码进行粗读之后，根据所理解的ARouter工作原理，对应使用流程，对原理进行解释，整理如下。

两条线：编译时 和 运行时

- 编译时：根据注解，生成路由表类
- 运行时：载入路由表，查找Path对应的Class

### arouter-compiler

三个Processors:

- RouteProcessor 普通路由注解
- AutowiredProcessor 参数注解
- InterceptorProcessor 拦截器注解

@Route & @Interceptor生成的路由表类，与 @Autowired 生成的参数注解类，名字生成规则不同，路径不同；

路由表固定路径：com.alibaba.android.arouter.routes
`ARouter$$Group$$GROUP_NAME.java`实现IRouteGroup接口，负责装载Group与其包含的Path；
`ARouter$$Root$$MODULE_NAME.java`实现IRouteRoot接口，负责装载Group与其对应的路由表（生成）类；
`ARouter$$Interceptor$$MODULE_NAME.java`实现IInterceptorGroup接口，负责装载Group与其对应的；

参数注解类动态路径，同注解此参数的组件路径：
`ACTIVITY_NAME$$ARouter$$Autowired.java` inject方法，负责从Intent中取出Extras并赋值给Activity中对应注解的变量。

### arouter-api

#### .init()

扫描固定路径下的apt文件，加载路由表到内存Map，缓存SP，对Root，Interceptor（除Group之外）的缓存表执行loadInto(...)，得到内存中的路由表Warehouse；

#### .build(...)

根据传入的path，提取group，产生一个新的Postcard（ARouter的基本操作单元，可理解为Intent）；

#### .with(...)

更新Postcard的Bundle；

#### .\_completion(postcard)

位于LogisticCenter中，非暴露方法。顾名思义。根据入参postcard的path，在内存路由表Warehouse中查找对应的RouteMeta（路由元数据）。如果不存在，找到对应的Group路由表执行loadInto()；然后完善完整的Postcard信息，包括type，destination，extras等。

#### .nagivation(...)

真正的nagivation之前会先执行\_completion(postcard)，如果没有异常，再走interceptorService.doInterceptions遍历所有（由InterceptorServiceImpl.java的逻辑可得知，其本身也是一个InterceptorService，通过@Route注册，在ARouter init时即完成初始化）的Interceptor，如果continue就调用\_navigation(...)，在这里是完成最终的Activity跳转（Intent，ActivityCompat.startActivity）或者Fragment实例返回。

#### .inject(activity)

装载对应activity的Autowired注册表，执行其中的inject方法。前面关于注解参数提到过，就是自动执行了getIntent，取出Extras并赋值给Activity中对应注解的变量。
