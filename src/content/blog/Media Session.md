---
pubDatetime: 2018-03-27
title: 如何构建一个Media App
featured: false
draft: false
tags:
  - Android
description: 本文主要是对Android开发者网站API Guide中“Media Apps”章节内容的翻译以及部分个人的理解。
---

在Android系统中构建一个具有多媒体功能的App，如果是使用系统的Media Player，那么就需要了解Android系统对Media的处理流程，会有很多的细节需要开发者关注，比如播放器的各种状态，物理按键的响应等。本文主要是对Android开发者网站API Guide中“Media Apps”章节内容的翻译以及部分个人的理解。

<!-- more -->

## Media APP 概览

### Player 和 UI

一个播放音视频的多媒体App通常包含两个部分：

1. 加载数字信息并呈现为音视频的播放器（player）；
2. 展示播放器状态和控制播放器控件的UI

![Multimedia APP](http://7u2h4k.com1.z0.glb.clouddn.com/15021978062336.png)
在Android中，可以选择系统提供的MediaPlayer，也可以使用其它第三方开源库如ExoPlayer来实现一个播放器。

### MediaSession 和 MediaController

UI的API和Player是相互独立的，两者之间的交互是所有多媒体App的本质；Android提供两个类：MediaSession 和 MediaController来支持这种结构。
MediaSession 和 MediaController之间通过定义的的和标准播放操作（play,pause,stop,etc.）相符合的callbacks来进行通信，也可以扩展出自定义的call来实现独特功能的app：
![MediaSession and MediaController](http://7u2h4k.com1.z0.glb.clouddn.com/15021984113212.png)

#### Media Session

Media Session负责与Player通信，对app的其它部分隐藏Player的操作，Player也只接受Media Session的控制。它管理着player当前播放的状态和具体信息。一个Media Session可以同时接收到多个Media Controller的callbacks，这也就是说为什么player可以被app的UI控制，也可以同时被其它运行Android Wear和Auto的设备控制。

#### Media Controller

App的UI只与Media Controller进行通信，它把控件操作（transport controls actions）转换成Media Session的callbacks，也可以在Media Session状态改变时接收session的callbacks，这就有了一个机制来保证关联UI自动更新。一个Media Controller一次只能连接到一个Media Session。

## Media Session

### 初始化

一个新创建的MediaSession必须要进行以下步骤的初始化工作：

1. 设置flags，使得MediaSession可以接受Media Controllers和Media buttons的Callbacks；
2. 创建并初始化一个`PlaybackStateCompat`的实例赋值给Session。播放状态的改变遍布Session，建议使用`PlaybackStateCompat.Builder`来复用；
3. 创建一个`MediaSessionCompat.Callback`的实例赋值给Session。

Media Session的创建和初始化工作应该在Activity或Service的`onCreate()`中进行。为了是media buttons在新启动（或者被停止）的app中能够起作用，`PlaybackState`必须在初始化的时候就包含`ACTION_PLAY`，这样才能匹配media buttons发送的Intent。（更多关于Media Button参见[Responding to Media Buttons](https://developer.android.google.cn/guide/topics/media-apps/mediabuttons.html)）

### 维护播放状态（Playback State）和元数据（metadata）

两个类可以代表media session的状态：1.`PlaybackStateCompat`描述了当前player的运行状态，包括：

- transport state（player是playing/paused/buffering，等）
- player position
- 当前状态可以处理的有效的controller actions

  2.`MediaMetadataCompat`代表了当前正在播放的内容：

- 艺术家&专辑&音轨 的名字
- 音轨时长
- 用于锁屏显示的专辑封面，最大320x320dp的bitmap

每当Playback state或者Metadata发生改变，都必须创建新的`PlaybackStateCompat.Builder()`或`MediaMetadataCompat.Builder()`实例，通过调用`setPlaybackState()`或者`setMetaData()`传递给Media session。为了在频繁操作的情况下减少内存的消耗，建议创建全局的builder对象，在整个media session中重用builder对象。

### 锁屏下的Media Session

从4.0（API 14）开始系统便可以访问一个media session的playback state和metadata，这也是为什么锁屏状态下可以显示当前播放的封面（Artwork）和控制器（Transport controls）。
在4.0及以上版本，如果metadata中包含这个专辑的artwork bitmap，就会会显示在锁屏状态的整个屏幕背景上；
在4.0（API 14）到4.4（API 19），当media session是活动状态且有artwork，那么同时也会自动显示Transport controls；而在5.0（API 21）及以上版本默认不再锁屏显示transport controls，需要使用[MediaStyle notification](https://developer.android.google.cn/guide/topics/media-apps/audio-app/building-a-mediabrowserservice.html#mediastyle-notifications)。

### Media session callbacks

Media session callback的主要方法是onPlay(), onPause(), and onStop()，在这些方法里添加控制Player的方法。
除了控制player和管理session状态切换，callbacks也起着控制app与其它app和设备硬件交互方式的作用。（参见[Handling Changes in Audio Output](https://developer.android.google.cn/guide/topics/media-apps/volume-and-earphones.html)）

## 创建一个Audio APP

一个音频app适用于典型的C/S架构。如下图：
![Audio app C/S](http://7u2h4k.com1.z0.glb.clouddn.com/15019227170401.png)
`MediaBrowserService`在这里有两个特点：

1. 当你使用MediaBrowserService，其它包含`MediaBrowser`的组件和应用都可以发现你的Service，创建它们自己的Controller，连接到你app的Media Session，然后控制Player。这也是Android Wear和Auto App获取访问Media App的方式。（补充：这也是为什么连接服务需要`onGetRoot`方法鉴定权限！）
2. 提供可选的Browsing API，使得client方可以访问Service然后创建自己的内容结构，可以是一个播放列表，也可以是一个媒体库或者精选集等（补充：这也即是`onLoadChildren`方法的作用）。

> Note：这里所指的MediaBrowserService和MediaBrowser在实现过程中推荐使用MediaBrowserServiceCompat和MediaBrowserCompat；MediaSession推荐使用MediaSessionCompat。

### 创建Media Browser Service

创建自己Service第一步是要新建一个类extends MediaBrowserServiceCompat，然后在APP的manifest中声明你自己的MediaBrowserService，必须包含一个特定的intent-filter。

```xml
<service android:name=".MediaPlaybackService">
 <intent-filter>
  <action android:name="android.media.browse.MediaBrowserService" />
 </intent-filter>
</service>
```

#### 初始化Media Session

在Service的onCreate()生命周期方法里需要完成以下工作：

1. 创建并初始化MediaSession
2. 设置MediaSession Callback
3. 设置MediaSession token

```java
public class MediaPlaybackService extends MediaBrowserServiceCompat {
  private MediaSessionCompat mMediaSession;
  private PlaybackStateCompat.Builder mStateBuilder;

  @Override
  public void onCreate() {
    super.onCreate();

    // Create a MediaSessionCompat
    mMediaSession = new MediaSessionCompat(context, LOG_TAG);

    // Enable callbacks from MediaButtons and TransportControls
    mMediaSession.setFlags(
      MediaSessionCompat.FLAG_HANDLES_MEDIA_BUTTONS |
      MediaSessionCompat.FLAG_HANDLES_TRANSPORT_CONTROLS);

    // Set an initial PlaybackState with ACTION_PLAY, so media buttons can start the player
    mStateBuilder = new PlaybackStateCompat.Builder()
                    .setActions(
                        PlaybackStateCompat.ACTION_PLAY |
                        PlaybackStateCompat.ACTION_PLAY_PAUSE);
    mMediaSession.setPlaybackState(mStateBuilder.build());

    // MySessionCallback() has methods that handle callbacks from a media controller
    mMediaSession.setCallback(new MySessionCallback());

    // Set the session's token so that client activities can communicate with it.
    setSessionToken(mMediaSession.getSessionToken());
  }
}

```

#### 管理client连接

MediaBrowserServiceCompat有两个方法：`onGetRoot()`控制service的访问；`onLoadChildren()`给client提供内容。

##### 通过onGetRoot()控制Client访问

该方法返回值（`BrowserRoot(@NonNull String rootId, @Nullable Bundle extras)`）为内容结构的根节点（root node of content hierarchy），如果返回null为拒绝访问。
如果要允许所有的clients访问service及获取内容，这里始终应该返回一个非空的、带有root ID的BrowserRoot；如果要仅允许连接service，不允许浏览内容，那么返回一个非空、但root ID为空的BrowserRoot。

```java
@Override
public BrowserRoot onGetRoot(String clientPackageName, int clientUid,
    Bundle rootHints) {

    // (Optional) Control the level of access for the specified package name.
    // You'll need to write your own logic to do this.
   if (allowBrowsing(clientPackageName, clientUid)) {
      // Returns a root ID, so clients can use onLoadChildren() to retrieve the content hierarchy
      return new BrowserRoot(MY_MEDIA_ROOT_ID, null);
    }
   else {
      // Clients can connect, but since the BrowserRoot is an empty string
      // onLoadChildren will return nothing. This disables the ability to browse for content.
      return new BrowserRoot("", null);
    }
}
```

##### 通过onLoadChildren()获取内容

client连接service成功之后就可以通过（可重复）调用`MediaBrowserCompat.subscribe()`来获取内容结构，进而展示到UI上。MediaBrowser的subscribe方法调用对应service的回调方onLoadChildren响应，得到一个`MediaBrowser.MediaItem`对象的列表。
每一个MeidaItem都有个唯一的ID（Demo中的id是通过对media的source uri进行hashcode得到的，现实中这个id可能是取自服务器方），当client想要打开或者播放一个item时会传入ID，service负责根据ID来取得对应的Item。

```java
@Override
public void onLoadChildren(final String parentMediaId,
    final Result<List<MediaItem>> result) {

  //  Browsing not allowed
  if (TextUtils.isEmpty(parentMediaId)) {
   result.sendResult(null);
   return;
  }

  // Assume for example that the music catalog is already loaded/cached.

  List<MediaItem> mediaItems = new ArrayList<>();

  // Check if this is the root menu:
  if (MY_MEDIA_ROOT_ID.equals(parentMediaId)) {

      // build the MediaItem objects for the top level,
      // and put them in the mediaItems list
  } else {

      // examine the passed parentMediaId to see which submenu we're at,
      // and put the children of that menu in the mediaItems list
  }
  result.sendResult(mediaItems);
}
```

Note：通过MediaBrowserService传递的MediaItem不应该直接包含icon bitmap，应该使用MediaDescription的setIconUri()来设置图片的Uri，使用到的时候再根据Uri去获取。

#### Media Browser Service生命周期

Android Service的行为表现取决于他是被启动（started）或者绑定到一个或多个客户端（bounded to one or more clients）。当一个Service被创建后，它可以被start，也可以bound，不管何种方式Service的具体任务不受影响，区别仅在于这个service可以存活多久。绑定的服务直到它所绑定的最后一个client被销毁之后才会被自动销毁，而启动的服务可以被显示的停止和销毁。
当一个运行在其它Activity中的MediaBrowser连接到MediaBrowserService时，即绑定了该Activity和Service，Service处于被绑定状态。这是集成在MediaBrowserServiceCompat中的默认操作。
一个仅仅处于被绑定状态是Service会在所有clients取消绑定后自动销毁。此例中UI activity 断开连接Service就会被销毁。在Audio App中，这显然不合理。用户期望可以一直听到音乐，无论是当前正在使用哪个app，activity有没有被回收。这就要求即使UI取消绑定，Service仍然不会被销毁，player还可以播放。
为此，需要在开始play之前，调用`startService()`来确保Service被启动。一个被启动的Service必须被显示的停止（无论是否存在绑定）。
可以调用`Context.stopService()`或`stopSelf()`来停止一个启动的service，系统会尽快的停止并回收它。如果仍然有client绑定这个service，停止和回收会被延迟到client取消绑定之后。
MediaBrowserService的生命周期取决于创建它的方式、绑定clients的数量，以及它所接收到的MediaSession callback。总结为以下：

1. 当为了响应Media button操作而启动，或者一个Activity绑定请求发生时，Service会被创建。
2. Media Session 的callback方法`onPlay()`中应该包含`startService()`，这样才能确保Service可以在所有的UI MediaBrowser activities取消绑定之后依然在存活。
3. Media Session 的callback方法`onStop()`中应该调用`stopSelf()`。

下面的图片展示了整个Service的生命周期（counter变量用来记录绑定数）：
![Service lifecycle flowchart](http://7u2h4k.com1.z0.glb.clouddn.com/15019310141694.png)

#### 在Foreground Service中使用MediaStyle notifications

首先解释一下Foreground Service。这里的Foreground是特殊意义的"前台”，是Android系统为了进程管理的目的把这个Service视为Foreground，而不是对于用户而言的屏幕可见的foreground（实际上Service始终都是工作在后台）。音乐Service正在播放，那么就应该是运行在foreground，系统就会知道当前service正在执行任务，就不会在内存紧张的时候结束服务。
当Service运行在foreground，就必须展示一个notification，最好还能有几个控制按钮，当然也应该展示Media Session metadata的一些基本信息。
在Player开始播放的时候创建并展示一条通知，最合适的位置就是在`MediaSessionCompat.Callback.onPlay()`方法里。
下面的示例代码展示了如何使用为Media App量身设计的`NotificationCompat.MediaStyle`，创建并展示metadata和控制按钮。使用`getController()`方法可以直接从media session中创建一个media controller对象。

```java
// Given a media session and its context (usually the component containing the session)
// Create a NotificationCompat.Builder

// Get the session's metadata
MediaControllerCompat controller = mediaSession.getController();
MediaMetadataCompat mediaMetadata = controller.getMetadata();
MediaDescriptionCompat description = mediaMetadata.getDescription();

NotificationCompat.Builder builder = new NotificationCompat.Builder(context);

builder
// Add the metadata for the currently playing track
    .setContentTitle(description.getTitle())
    .setContentText(description.getSubtitle())
    .setSubText(description.getDescription())
    .setLargeIcon(description.getIconBitmap())

// Enable launching the player by clicking the notification
    .setContentIntent(controller.getSessionActivity())

// Stop the service when the notification is swiped away
    .setDeleteIntent(MediaButtonReceiver.buildMediaButtonPendingIntent(this,
       PlaybackStateCompat.ACTION_STOP))

// Make the transport controls visible on the lockscreen
    .setVisibility(NotificationCompat.VISIBILITY_PUBLIC)

// Add an app icon and set its accent color
// Be careful about the color
    .setSmallIcon(R.drawable.notification_icon)
    .setColor(ContextCompat.getColor(this, R.color.primaryDark))

// Add a pause button
      .addAction(new NotificationCompat.Action(
          R.drawable.pause, getString(R.string.pause),
          MediaButtonReceiver.buildMediaButtonPendingIntent(this,
              PlaybackStateCompat.ACTION_PLAY_PAUSE)))

// Take advantage of MediaStyle features
    .setStyle(new NotificationCompat.MediaStyle()
      .setMediaSession(mediaSession.getSessionToken())
      .setShowActionsInCompactView(0)
// Add a cancel button
      .setShowCancelButton(true)
      .setCancelButtonIntent(MediaButtonReceiver.buildMediaButtonPendingIntent(this,
          PlaybackStateCompat.ACTION_STOP));

// Display the notification and place the service in the foreground
startForeground(id, builder.build());
```

（此处还有一些关于MediaStyle的详细介绍，不再展开，参见官方英文原文）

### 创建Media Browser Client

为了完成这个C/S结构，还必须要有一个Activity UI，一个MediaController，以及MediaBrowser。MediaBrowser扮演了两个角色：连接MediaBrowserService，并在这个链接上为UI创建一个MediaController；说白了就是桥梁。

#### 连接MediaBrowserService

在Activity创建的时候进行Service连接操作，这里有一些握手操作（Activity的生命周期Callback中）需要注意：

1. `onCreate()`构造MediaBrowserCompat，传入定义的MediaBrowserService，以及MediaBrowserCompat.ConnectionCallback。
2. `onStart()`连接MediaBrowserService，这里也正是MediaBrowserCompat.ConnectionCallback魔法发生的地方：如果连接成功，`onConnected()`回调中创建media controller，并将之关联到media session，连接UI controls与media controller，然后注册controller以收到media session callback回调。（魔法已内置，无需手动）
3. `onStop()`断开MediaBrowser连接，取消注册MediaController.Callback。

```java
public class MediaPlayerActivity extends AppCompatActivity {
  private MediaBrowserCompat mMediaBrowser;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // ...
    // Create MediaBrowserServiceCompat
    mMediaBrowser = new MediaBrowserCompat(this,
      new ComponentName(this, MediaPlaybackService.class),
        mConnectionCallbacks,
        null); // optional Bundle
  }

  @Override
  public void onStart() {
    super.onStart();
    mMediaBrowser.connect();
  }

  @Override
  public void onStop() {
    super.onStop();
    // (see "stay in sync with the MediaSession")
    if (MediaControllerCompat.getMediaController(MediaPlayerActivity.this) != null) {
      MediaControllerCompat.getMediaController(MediaPlayerActivity.this).unregisterCallback(controllerCallback);
    }
    mMediaBrowser.disconnect();

  }
}
```

> Note：这里仅是已Activity做为UI来举例，具体实现中换成Fragment的逻辑与上述一致。

#### 定制MediaBrowserCompat.ConnectionCallback

Activity构造完MediaBrowserCompat之后，然后就需要创建一个ConnectionCallback的实例，在`onConnected()`回调中获取Media Session的Token，并用这个token去创建MediaControllerCompat，然后用`MediaControllerCompat.setMediaController()`来保存一个UI与controller的连接。

```java
private final MediaBrowserCompat.ConnectionCallback mConnectionCallbacks =
  new MediaBrowserCompat.ConnectionCallback() {
    @Override
    public void onConnected() {

      // Get the token for the MediaSession
      MediaSessionCompat.Token token = mMediaBrowser.getSessionToken();

      // Create a MediaControllerCompat
      MediaControllerCompat mediaController =
        new MediaControllerCompat(MediaPlayerActivity.this, // Context
        token);

      // Save the controller
      MediaControllerCompat.setMediaController(MediaPlayerActivity.this, mediaController);

      // Finish building the UI
      buildTransportControls();
    }

    @Override
    public void onConnectionSuspended() {
      // The Service has crashed. Disable transport controls until it automatically reconnects
    }

    @Override
    public void onConnectionFailed() {
      // The Service has refused our connection
    }
  };
```

#### 连接UI与Media controller

在UI上通过MediaControllerCompat.TransportControls 方法来控制controller。

#### 与Media Session同步

UI理应展示media session的最新状态，包括PlaybackState与Metadata。当你创建transport contraols时你可以获取到当前session的状态，来对应调整ui以及controls的可以操作等；创建之后，就需要一个来自Media Session的callback来获取状态的改变了，它就是`MediaControllerCompat.Callback`。这个回调也应当在onConnected之后注册到controller。

### Media Session Callbacks

在media session callback中要调用许多的API，去控制Player，管理audio focus，管理session与media browser service的通信等。下表总结了这些工作在callbacks中如何分布。
![screencapture-developer-android-google-cn-guide-topics-media-apps-audio-app-mediasession-callbacks-html-1502180848097](http://7u2h4k.com1.z0.glb.clouddn.com/screencapture-developer-android-google-cn-guide-topics-media-apps-audio-app-mediasession-callbacks-html-1502180848097.png)

## 响应Media Buttons

这里的buttons包含且不仅限于Android设备上的物理按钮、有线/蓝牙耳机上的按钮、其他周边设备按钮。用户的点击按钮操作会在Android上产生一个包含标识的KeyEvent，key code以`KEYCODE_MEDIA`开头（如KEYCODE_MEDIA_PLAY）。

Android系统分发Media button Event规则：

1. 首先分发给当前屏幕显示的Activity（foreground activity）；
2. 如果当前Activity没有处理，系统会尝试发送给一个活动状态的MediaSession（调用`setActive(true)后`。如果有多个活动的MediaSession，系统会优先选择状态为准备播放（buffering/connecting)、播放中（playing）或者暂停（paused），而不会是停止（stopped）。
3. 如果没有活动状态的MediaSession，系统会尝试发送给最近一次活动的MediaSession。在5.0（API21）及以上则是发送给调用了`setMediaButtonReceiver()`方法的Session。

由于系统版本的割裂，在不同版本上也有不同的版本的处理方法，这里仅对方案总结如下：

- **通用**：

  - 在初始化时对MediaSession设置标签：

  ```java
  mediaSession.setFlags(MediaSessionCompat.FLAG_HANDLES_MEDIA_BUTTONS);
  ```

  -     在Service的`onStartCommand()`中添加代码（这里MediaButtonReceiver的作用是解释intent并生成对应MediaSession的callbak，onPlay onPause等）：

  ```java
   public int onStartCommand(Intent intent, int flags, int startId) {
     MediaButtonReceiver.handleIntent(mMediaSessionCompat, intent);
     return super.onStartCommand(intent, flags, startId);
   }
  ```

- **5.0及以上**：

  - 在MediaController的callback方法`onConnected()`中调用`MediaControllerCompat.setMediaController()`（交由系统默认处理）；
  - 如果需要允许Media Button的Event重新启动非活动状态的Media Session，手动调用`setMediaButtonReceiver(PendingIntent intent)`。

- **5.0以下**：

  - 在Activity中override `onKeyDownEvent()`以接收处理Media buttons event（必须return true，标识event已被处理）：

  ```java
  @Override
  boolean onKeyDown(int keyCode, KeyEvent event) {
              switch (keyCode) {
              case KeyEvent.KEYCODE_MEDIA_PLAY:
                      yourMediaController.dispatchMediaButtonEvent(event);
                      return true;
              }
              return false;
      }
  ```

  - 在Manifest文件中声明全局的`MediaButtonReceiver`：

  ```xml
  <receiver android:name="android.support.v4.media.session.MediaButtonReceiver" >
     <intent-filter>
       <action android:name="android.intent.action.MEDIA_BUTTON" />
     </intent-filter>
   </receiver>
  ```

## 处理音频输出中的变化

除了要响应UI Controls和Media Button，一个音频App还需要对其它可能影响到声音的Android事件做出响应，主要有以下三种：

1. 当用户通过点击物理按钮改变音量时对应调整音量；
2. 当正在使用中的耳机断开连接时暂停播放；
3. 当其它应用拿到了音频输出流时停止播放或降低音量。

### 响应音量控制按钮

Android对不同的用途使用不同的音频流（Audio Stream），播放音乐，闹钟，通知，来电铃声，系统声音，通话音量等。用户可以独立的控制每一个stream的音量。默认情况下，按下音量控制按钮会改变当前活动状态的音频流，如果当前没有任何正在播放，就调整铃声音量。
除非你的app是一个闹钟程序，否则都应该使用`STREAM_MUSIC`来播放音频。

```java
setVolumeControlStream(AudioManager.STREAM_MUSIC);
```

这是一个Activity方法，最好是在onCreate()中就调用，这样当Activity或Fragment可见时，音量按钮就可以连接上STREAM_MUSIC。

### 不要太吵

当有线耳机被拔掉，或者蓝牙耳机断开连接时，音频流会自动切换到内置扬声器。如果你正在以一个很高的音量听音乐，那这就很吵很尴尬了。
好在，当以上情况发生时，系统会发出一条`ACTION_AUDIO_BECOMING_NOISY`intent广播，创建一个Receiver接收这条广播，在回调中控制暂停或者降低音量：

```java
private class BecomingNoisyReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
      if (AudioManager.ACTION_AUDIO_BECOMING_NOISY.equals(intent.getAction())) {
          // Pause the playback
      }
    }
}
```

在开始播放时注册Receiver，在停止时取消注册。按照指导规范，对应的是MediaSession Callbacks的onPlay()和onStop()。

```java
private IntentFilter intentFilter = new IntentFilter(AudioManager.ACTION_AUDIO_BECOMING_NOISY);
private BecomingNoisyReceiver myNoisyAudioStreamReceiver = new BecomingNoisyReceiver();

MediaSessionCompat.Callback callback = new
MediaSessionCompat.Callback() {
  @Override
  public void onPlay() {
    registerReceiver(myNoisyAudioStreamReceiver, intentFilter);
  }

  @Override
  public void onStop() {
    unregisterReceiver(myNoisyAudioStreamReceiver);
  }
}
```

### 共享Audio Focus

为了避免多个App同时播放造成混乱，Android引入音频焦点（Audio Focus）的概念，在一个时间点最多只有一个App可以拥有焦点。

一个规范的音频App应当遵循以下规则来管理音频焦点：

1. 开始播放之前，请求焦点，验证是否授予成功；
2. 当其它app获得焦点，停止播放或者降低音量播放；
3. 停止播放时，释放焦点。

以上原则仅为从用户体验角度来鼓励遵照，但也不强制。

#### 获取和释放焦点

在进行播放之前，Media Session的onPlay()回调方法中调用`requestAudioFocus()`并验证`AUDIOFOCUS_REQUEST_GRANTED`是否成功：

```java
AudioManager am = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
AudioManager.OnAudioFocusChangeListener afChangeListener;

...
// Request audio focus for playback
int result = am.requestAudioFocus(afChangeListener,
                             // Use the music stream.
                             AudioManager.STREAM_MUSIC,
                             // Request permanent focus.
                             AudioManager.AUDIOFOCUS_GAIN);

if (result == AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {
    // Start playback
}
```

参数1 **AudioManager.OnAudioFocusChangeListener** 焦点变化回调，应该创建在拥有Media Session的Activity或Service中，下个小节展开。

参数3 **duration hint**，指定请求焦点的使用范围：

- `AUDIOFOCUS_GAIN`永久焦点，在可预见的未来一直播放，期望上一个焦点应用停止播放；
- `AUDIOFOCUS_GAIN_TRANSIENT`暂时焦点，预计短时间播放，期望上一个焦点应用暂停播放；
- `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK`带‘DUCK’的暂时焦点，预计短时播放，且不需要上一个焦点应用暂停或停止，可以降低音量同时播放（Duck means Lower）。

```java
AudioManager am = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
AudioManager.OnAudioFocusChangeListener afChangeListener;

...
// Request audio focus for playback
int result = am.requestAudioFocus(afChangeListener,
                             // Use the music stream.
                             AudioManager.STREAM_MUSIC,
                             // Request permanent focus.
                             AudioManager.AUDIOFOCUS_GAIN);

if (result == AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {
    // Start playback
}
```

播放结束，请求释放焦点：

```java
// Abandon audio focus when playback complete
am.abandonAudioFocus(afChangeListener);
```

#### 响应音频焦点变化

一个请求音频焦点的app必须要在其它app请求焦点的时候可以自己释放焦点。这就是AudioManager.OnAudioFocusChangeListener的意义所在。
如下代码所示，参数**focusChange**指正在发生的变化，也就是正在请求获取焦点的app所指定的duration hint，当前app应当对应的做出响应：

```java
private Handler mHandler = new Handler();
AudioManager.OnAudioFocusChangeListener afChangeListener =
  new AudioManager.OnAudioFocusChangeListener() {
    public void onAudioFocusChange(int focusChange) {
      if (focusChange == AudioManager.AUDIOFOCUS_LOSS) {
        // Permanent loss of audio focus
        // Pause playback immediately
        mediaController.getTransportControls().pause();
        // Wait 30 seconds before stopping playback
        mHandler.postDelayed(mDelayedStopRunnable,
          TimeUnit.SECONDS.toMillis(30));
      }
      else if (focusChange == AUDIOFOCUS_LOSS_TRANSIENT) {
        // Pause playback
      } else if (focusChange == AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK) {
        // Lower the volume, keep playing
      } else if (focusChange == AudioManager.AUDIOFOCUS_GAIN) {
        // Your app has been granted audio focus again
        // Raise volume to normal, restart playback if necessary
      }
    }
  };
```

```java
private Runnable mDelayedStopRunnable = new Runnable() {
    @Override
    public void run() {
        mediaController.getTransportControls().stop();
    }
};
```

为了确保用户重启播放时，延时停止操作不会发生，必须要在任意状态变化响应时调用`mHandler.removeCallbacks(mDelayedStopRunnable)`。

## 参考

- [开发者文档 Media Apps](https://developer.android.google.cn/guide/topics/media-apps/media-apps-overview.html)
