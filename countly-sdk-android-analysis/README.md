countly-sdk-android 源码解析
====================================
>  项目地址：[countly-sdk-android](https://github.com/Countly/countly-sdk-android)，分析的版本：[16.02.01 Release](https://github.com/Countly/countly-sdk-android/commit/c105d06e573703d9e29d5c92068a41befa36f43f)，Demo 地址：[countly-sdk-android-demo](https://github.com/Labmem003/zhenzhi-open-project-analysis/tree/master/countly-sdk-android-demo)    
 分析者：[振之](https://github.com/Labmem003)，分析状态：未完成

工作需要用到了一个叫Countly的开源移动应用实时统计分析系统，类似于友盟。挺好奇这类统计系统的实现机制的，于是读了一下源码。项目其实十分简单，功能性代码比较多。原理上并没有什么特别高深莫测的地方，跟我们平常写业务代码的套路是一样一样的。

笔记如下，有需要／好奇的可以看看。

读完大概需要15分钟。
###1. 功能介绍  
####1.1 Countly
Countly 是一款类似于友盟的移动&Web应用通用的实时统计分析系统，专注于易用性、扩展性和功能丰富程度。不同之处是Countly是开源的，任何人都可以将Countly客户端部署在自己的服务器并将开发工具包整合到他们的应用程序中。比友盟要简洁干净，关键是数据和程序都完全处于自己掌控之下，不愿被第三方掌握数据，或者有什么特殊需求的，可以自己满足自己了。

这里分析的是其Android端的sdk, 以了解和学习移动应用统计类的工具收集App的使用情况和终端用户的行为的机制。主要功能包括App基本数据收集、自定义事件记录、崩溃报告。
####1.2 基本使用
（1）将 Countly SDK 添加到您的项目
Android Studio
添加二进制 Maven 存储库：

```
buildscript {
    repositories {
        maven {
            url  "http://dl.bintray.com/countly/maven"
        }
    }
}
```
添加 Countly SDK 依赖：

```
dependencies {
    compile 'ly.count.android:sdk:15.06'
}
```
当然也可以直接下载Jar使用, 这里分析的是[sdk-16.02.01.jar](https://github.com/Countly/countly-sdk-android/releases/download/16.02.01/sdk-16.02.01.jar)。

（2）服务器端程序的安装和使用
篇幅和重点所限，不详细介绍服务端安装使用，在[Countly体验](https://cloud.count.ly)创建一个应用试用即可。

（3）具体使用

设置SDK，AppKey在（2）的“管理－应用”中查看: 

```
Countly.sharedInstance().init(this, "https://YOUR_SERVER", "YOUR_APP_KEY");
```
记录事件：

```
Countly.sharedInstance().recordEvent("purchase", 1);
```

设置崩溃报告：

```
Countly.sharedInstance().enableCrashReporting()
```
更详细的使用，可参照我写的小[Demo](https://github.com/Labmem003/zhenzhi-open-project-analysis/tree/master/countly-sdk-android-demo)。
###2. 总体设计
上面是Countly SDK的总体设计图。

SDK主要处理Event、Crash和会话流（Session）3种数据记录请求。其中Crash和Session自动记录，并作为Connection持久存储到ConnectionQueue, 等待提交到服务器；Event则由开发者调用，并配有一个EventQueue存储，但是在上报给服务器的时候依然是通过加入到ConnectionQueue。也就是说，所有请求，最后都是Connection。
ConnectionQueue和EventQueue不是平常意义的FIFO队列，而是本地存储队列。包装了基于SharePreference实现的持久层Store，每个请求会被字符串化，加上分隔符，添加到对应的SP键值后面。

最终存储在SP的ConnectionQueue，大概长这样：

```
"app_key=appKey_&timestamp=3482759874&hour=6&dow=2&session_duration=24&location=3，8:::app_key=appKey_&timestamp=345567773&hour=8&dow=3&session_duration=12&location=3，8"
```
OK, 接口地址知道，数据在手, 取出来按接口要求拼装好，fire the hole 就是了。

###3. 流程图
主要功能流程图

###4. 详细设计
###4.1 类关系图
  
###4.2 类详细介绍
结构很简单，一共就两个包，countly核心包和openudid包。
上图。
countly包解决统计什么，怎么实施统计；而openudid包解决如何标记统计的数据来自何方。

####4.2.1 openudid包
先来看看比较简单的openudid包，她是一个设备标识方案，能提供一个设备通用统一标识符（Unique Device IDentifier/UDID）。如果同一台设备上有多个App都用了这个包来生成UDID，他们获取的UDID是一致的（即所谓设备标识）。当我们将统计数据发送给服务端时，会将UDID附上。不难想到，之后服务端算日活、描述用户特征、事件追踪等等各种后续的数据分析肯定都离不开UDID，算是很必要的基础设施。实际上她也是一个开源包[OpenUDID](https://github.com/vieux/OpenUDID)。
#####4.2.1.1 OpenUDID_service.java
这个类很简单，就是一个只重写了onTransact方法的Service，支持跨进程调用。

```
public class OpenUDID_service extends Service{
	@Override
	public IBinder onBind(Intent arg0) {
		return new  Binder() {
			@Override
			public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) {
				final SharedPreferences preferences = getSharedPreferences(OpenUDID_manager.PREFS_NAME, Context.MODE_PRIVATE);
		
				reply.writeInt(data.readInt()); //Return to the sender the input random number
				reply.writeString(preferences.getString(OpenUDID_manager.PREF_KEY, null));
				return true;
			}
		};
	}
}
```
它给调用者返回了两个值：

（1）把调用者传过来的data中的int参数又传回去了，起的是标识发起者的作用，类似与startActivityForResult()中的requestCode。

```
reply.writeInt(data.readInt()); //Return to the sender the input random number
```

（2）返回本进程获取到的OpenUDID。还可以看到 OpenUDID 是用SharePreferences保存的。

```
reply.writeString(preferences.getString(OpenUDID_manager.PREF_KEY, null));

```
(面大家解释的都比较清楚了。我想补充的是，我们需要唯一标识设备，还是标识app? 按照apple的政策，标识设备是不可以的，所以UDID，MAC都被禁止访问。我们要统计用户怎么办？OK标识app是可以的，用UUID可以实现，我理解OpenUDID就是一种UUID。可以被重新生成的ID。
单纯OpenUDID的代码是不会被拒，但获取MAC地址的方法是会被拒的。
根据你的解释，那我的理解，关于OpenUDID，如果一个开发者的一个App自己想识别设备，还是可以用OpenUDID的吧？对么？

http://blog.woodbunny.com/post-104.html
http://blog.csdn.net/meegomeego/article/details/16858883

https://www.zhihu.com/question/20141418

讲讲苹果禁掉udid的理由，佩服一下。
现在ios9都出来了，应该有解决方案了吧，这里分析android端为主，不过多关注ios的情况。

UDID与OpenUDID的不同之处
每台iOS设备的UDID是唯一且永远不会改变；
每台iOS设备的OpenUDID是通过第一个带有OpenUDID SDK包的App生成，如果你完全删除全部带有OpenUDID SDK包的App（比如恢复系统等），那么OpenUDID会重新生成，而且和之前的值会不同，相当于新设备；
是否足够替代
普通的iOS设备用户不会没事就去恢复系统或者抹掉系统，所以一般OpenUDID的值是不会改变的；
在iOS系统升级换代时，会产生较大的影响，毕竟95%以上的iOS设备用户都会选择升级到最新的系统；
是否足够替代就看你对UDID的需求是什么了，如果要求怎么都不能变，那OpenUDID可能还是不能满足你的需求！)
#####4.2.1.2 OpenUDID_manager.java
调用sync(Context context)来初始化一个udid, 策略是先看自己有木有，木有的话就从好基友那里拿，好基友也没有就只能自己撸一个出来。

一切都撸完之后，调用getOpenUDID就可以得到一枚UDID了。

（1）public static void sync(Context context)
 
 ```
 //先尝试从本地SharePreference中获取
		OpenUDID = manager.mPreferences.getString(PREF_KEY, null);
```
```
		if (OpenUDID == null) //本地没有
		{
//获取设备上所有的使用了该包的OpenUDID_service的services列表,intent形式保存在mMatchingIntents
			manager.mMatchingIntents = context.getPackageManager().queryIntentServices(new Intent("org.OpenUDID.GETUDID"), 0);
						
			if (manager.mMatchingIntents != null)
				//尝试从别的使用了OpenUDID_service的进程获取
				manager.startService();
		
		} else {//本地存在UDID, 初始化完成，即可以直接调用getOpenUDID()来获取了
			mInitialized = true;
		}
 ```


(2)startService()
启动mMatchingIntents的首个intent并移除；

 ```
 private void startService() {
		if (mMatchingIntents.size() > 0) { //There are some Intents untested
			if (LOG) Log.d(TAG, "Trying service " + mMatchingIntents.get(0).loadLabel(mContext.getPackageManager()));
		
			final ServiceInfo servInfo = mMatchingIntents.get(0).serviceInfo;
            final Intent i = new Intent();
            i.setComponent(new ComponentName(servInfo.applicationInfo.packageName, servInfo.name));
            mMatchingIntents.remove(0);
            try	{	// try added by Lionscribe
            	mContext.bindService(i, this,  Context.BIND_AUTO_CREATE);
            }
            catch (SecurityException e) {
                startService();	// ignore this one, and start next one
            }
		} else { //No more service to test
			
			getMostFrequentOpenUDID(); //Choose the most frequent
	
			if (OpenUDID == null) //No OpenUDID was chosen, generate one			
				generateOpenUDID();
			if (LOG) Log.d(TAG, "OpenUDID: " + OpenUDID);

			storeOpenUDID();//Store it locally
			mInitialized = true;
		}
	}
```
若启动成功则可以拿到被启动进程的UDID,以UDID为键，次数为值存入HashMap中；然后再次startService();结果是递归地启动了mMatchingIntents的所有intent，得到一张记录着各个进程的不相同的udid及其次数的Map。
	
```
	public void onServiceConnected(ComponentName className, IBinder service) {
		//Get the OpenUDID from the remote service
		...
				final String _openUDID = reply.readString();
				...

					if (mReceivedOpenUDIDs.containsKey(_openUDID)) mReceivedOpenUDIDs.put(_openUDID, mReceivedOpenUDIDs.get(_openUDID) + 1);
					else mReceivedOpenUDIDs.put(_openUDID, 1);
						...
		startService(); //Try the next one
  	}
 ```
（3）private void getMostFrequentOpenUDID() ，返回Map中次数最多的UDID

（4）private void generateOpenUDID() ，没有从其他进程领到UDID，就生成一个
 
 ```
 	/*
	 * Generate a new OpenUDID
	 */
	private void generateOpenUDID() {
		if (LOG) Log.d(TAG, "Generating openUDID");
		//Try to get the ANDROID_ID
		OpenUDID = Secure.getString(mContext.getContentResolver(), Secure.ANDROID_ID); 
		if (OpenUDID == null || OpenUDID.equals("9774d56d682e549c") || OpenUDID.length() < 15 ) {
			//if ANDROID_ID is null, or it's equals to the GalaxyTab generic ANDROID_ID or bad, generates a new one
			final SecureRandom random = new SecureRandom();
			OpenUDID = new BigInteger(64, random).toString(16);
		}
```  
直接使用了Android ID来做UDID，其实有点不靠谱。因为如果你恢复了出厂设置，那他就会改变的。而且如果你root了手机，你也可以改变这个ID。不过如果不需要太精确的统计，也够用了。看你的需求吧。

可以参考这篇文章来修改适合你的UDID。
http://www.bkjia.com/Androidjc/1036506.html

(5)public static String getOpenUDID(),就是getOpenUDID。

####4.2.2 countly包
概念解释

Event

Session

Crash

Connection
#####4.2.2.1 OpenUDIDAdapter.java
包装了UDID包，提供sync（），getOpenUDID（）。但是是用动态反射的方法封装的，不明白为什么。官方的commit 木essage说了一句：call OpenUDID dynamically so that including the OpenUDID source is not necessary to get the Countly Android SDK to work when an app provides it's own deviceID。
看懂的请告诉我。
#####4.2.2.2 DeviceId.java
代表设备ID的类。 
（1）主要属性是

private String id;//设备标识ID

private Type type;//ID类型

（2）ID类型分为3种

```
public static enum Type {
        DEVELOPER_SUPPLIED,//开发者自行定义和提供
        OPEN_UDID,//openUDID
        ADVERTISING_ID,//谷歌广告平台设备标识符
    }
```
openUDID前面已介绍过，其他两种也是类似，不累述。
#####4.2.2.3 DeviceInfo.java
一个纯POJO类，用来存放设备信息，如设备名称、设备分辨率、版本号等。

主要方法：
static String getMetrics(final Context context)，返回url-encoded的属性json字符串。
#####4.2.2.4 CrashDetails.java
提供了一些静态方法来获取运行时环境信息，结合DeviceInfo类, 为Crash时提供详细的参考信息。

主要方法：
列举，json生成。

```
 static String getCrashData(final Context context, String error, Boolean nonfatal) {
        final JSONObject json = new JSONObject();

        fillJSONIfValuesNotEmpty(json,
                "_error", error,
                "_nonfatal", Boolean.toString(nonfatal),
                "_logs", getLogs(),
                "_device", DeviceInfo.getDevice(),
                "_os", DeviceInfo.getOS(),
                "_os_version", DeviceInfo.getOSVersion(),
                "_resolution", DeviceInfo.getResolution(context),
                "_app_version", DeviceInfo.getAppVersion(context),
                "_manufacture", getManufacturer(),
                "_cpu", getCpu(),
                "_opengl", getOpenGL(context),
                "_ram_current", getRamCurrent(context),
                "_ram_total", getRamTotal(context),
                "_disk_current", getDiskCurrent(),
                "_disk_total", getDiskTotal(),
                "_bat", getBatteryLevel(context),
                "_run", getRunningTime(),
                "_orientation", getOrientation(context),
                "_root", isRooted(),
                "_online", isOnline(context),
                "_muted", isMuted(context),
                "_background", isInBackground()
                );

       ...
            json.put("_custom", getCustomSegments());
        ...
        String result = json.toString();

        ...
            result = java.net.URLEncoder.encode(result, "UTF-8");
        ...

        return result;
    }
```
#####4.2.2.5 UserData.java
类似CrashDetail,简单类。

```
/*
     * Send provided values to server
     */
    public void save(){
        connectionQueue_.sendUserData();
        UserData.clear();
    }
```
#####4.2.2.6 Event.java
定义了一个事件的数据结构

```
	public String key;//键，识别事件
    public int count;//发生此事件的次数
    public double sum;//事件的全部数值数据，比如一次支付事件的支付金额，可选
	public Map<String, String> segmentation;//分段键值对，用来扩展自定义数据，数量不受限制
```
由于多个事件可结合在单一请求中，为了正确报告和处理数据（特别是排队数据），还有下面3个属性用来提供数据记录时间：

```
	public int timestamp;//时间戳
    public int hour;//本地时间，0-23
    public int dow;//星期几
```    

```
JSONObject toJSON()
static Event fromJSON(final JSONObject json)
```
还有fromJson和toJson函数来在事件对象和json表示之间转换。这个类就这些了。
#####4.2.2.7 EventQueue.java
这个类用来队列化event，并且可以将event转化为json，方便提交到服务器。

（1）主要属性：

```
private final CountlyStore countlyStore_;
```
CountlyStore是一个持久化存储类，EventQueue类其实就是对CountlyStore的一个封装，每次有Event添加进queue，就会通过CountlyStore直接持久化存储到本地（本地队列的末尾）；出queue的时候也是直接从本地队列中移除。CountlyStore在后面还会详细介绍。

```
    /**
     * Constructs an EventQueue.
     * @param countlyStore backing store to be used for local event queue persistence
     */
    EventQueue(final CountlyStore countlyStore) {
        countlyStore_ = countlyStore;
    }
```

（2）主要方法：
并不是真正意义上的queue，使用recordEvent入队一个Event到本地，使用events直接把整个队列提取出来并且转换为urlEncoded的json串，可以直接用于提交给服务器。好处是合并请求，方便一次提交多个event，当然也是因为配合api。

```
String events() {
        String result;
	//从CountlyStore取回EventQueue
        final List<Event> events = countlyStore_.eventsList();
        
        //Json化
        final JSONArray eventArray = new JSONArray();
        for (Event e : events) {
            eventArray.put(e.toJSON());
        }

        result = eventArray.toString();

        countlyStore_.removeEvents(events);

	//UrlEncode
        try {
            result = java.net.URLEncoder.encode(result, "UTF-8");
        } catch (UnsupportedEncodingException e) {
            // should never happen because Android guarantees UTF-8 support
        }

        return result;
    }
    
```

```
void recordEvent(final String key, final Map<String, String> segmentation, final int count, final double sum) {
        ...
        //直接持久化存入本地Event队列
        countlyStore_.addEvent(key, segmentation, timestamp, hour, dow, count, sum);
    }
```
#####4.2.2.8 CountlyStore.java
该类使用SharePreference为Event,Connection,Location 3类数据提供持久化服务.
介绍存储方式。

以Event为例，每次有新来的Event，会先读取本地的events, 把新来的event加到末尾，然后整个队列重新被json化存储到Shareference.

```
void addEvent(final Event event) {
        final List<Event> events = eventsList();
        events.add(event);
        preferences_.edit().putString(EVENTS_PREFERENCE, joinEvents(events, DELIMITER)).commit();
    }
```

直接从SharePreference取出所有json表示的events

```
public String[] events()
```

将json表示的events反序列化，并排序，返回。

```
public List<Event> eventsList() {
        final String[] array = events();
        final List<Event> events = new ArrayList<>(array.length);
        for (String s : array) {
            try {
                final Event event = Event.fromJSON(new JSONObject(s));
                if (event != null) {
                    events.add(event);
                }
            } catch (JSONException ignored) {
                // should not happen since JSONObject is being constructed from previously stringified JSONObject
                // events -> json objects -> json strings -> storage -> json strings -> here
            }
        }
        // order the events from least to most recent
        Collections.sort(events, new Comparator<Event>() {
            @Override
            public int compare(final Event e1, final Event e2) {
                return e1.timestamp - e2.timestamp;
            }
        });
        return events;
    }
```

```
 public String[] connections()
 public String[] events()
 public List<Event> eventsList()
```

#####4.2.2.9 Countly.java
暴露接口和驱动各个类工作的入口类。主要的属性是
EventQueue，ConnectionQueue，和ScheduledExecutorService。
（1）构造和init
调用其他接口前需先调用init()来初始化，主要是参数检验、初始化和配置EventQueue，ConnectionQueue。构造的时候就开启了一个1分钟1次的定时任务。

```
 Countly() {
        connectionQueue_ = new ConnectionQueue();
        Countly.userData = new UserData(connectionQueue_);
        timerService_ = Executors.newSingleThreadScheduledExecutor();
        timerService_.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                onTimer();
            }
        }, TIMER_DELAY_IN_SECONDS, TIMER_DELAY_IN_SECONDS, TimeUnit.SECONDS);
    }
...
  deviceIdInstance.init(context, countlyStore, true);

            connectionQueue_.setServerURL(serverURL);
            connectionQueue_.setAppKey(appKey);
            connectionQueue_.setCountlyStore(countlyStore);
            connectionQueue_.setDeviceId(deviceIdInstance);

            eventQueue_ = new EventQueue(countlyStore);
            ...
```

（2）定时任务
记录session和event到connectionqueue，并触发发送数据行为，所有的数据都是从connectionqueue 的store中取得。

```
synchronized void onTimer() {
        final boolean hasActiveSession = activityCount_ > 0;
        if (hasActiveSession) {
            if (!disableUpdateSessionRequests_) {
                connectionQueue_.updateSession(roundedSecondsSinceLastSessionDurationUpdate());
            }
            if (eventQueue_.size() > 0) {
                connectionQueue_.recordEvents(eventQueue_.events());
            }
        }
    }
```

(3)Session更新，利用activityCount
（4）异常捕获,使用UncaughtExceptionHandler

```
 /**
     * Enable crash reporting to send unhandled crash reports to server
     */
    public synchronized Countly enableCrashReporting() {
        //get default handler
        final Thread.UncaughtExceptionHandler oldHandler = Thread.getDefaultUncaughtExceptionHandler();

        Thread.UncaughtExceptionHandler handler = new Thread.UncaughtExceptionHandler() {

            @Override
            public void uncaughtException(Thread t, Throwable e) {
                StringWriter sw = new StringWriter();
                PrintWriter pw = new PrintWriter(sw);
                e.printStackTrace(pw);
                Countly.sharedInstance().connectionQueue_.sendCrashReport(sw.toString(), false);

                //if there was another handler before
                if(oldHandler != null){
                    //notify it also
                    oldHandler.uncaughtException(t,e);
                }
            }
        };

        Thread.setDefaultUncaughtExceptionHandler(handler);
        return this;
    }
```
（5）记录事件

事件先被记录在eventQueue, 到了需要被发送的时候会被全部取出放入connectionQueue。connectionQueue有发送数据的能力，系统定时从把connectionQueue中的多条数据合并发送到服务器。

```
public synchronized void recordEvent(final String key, final Map<String, String> segmentation, final int count, final double sum) {
       ...
        eventQueue_.recordEvent(key, segmentation, count, sum);
        sendEventsIfNeeded();
    }
    
void sendEventsIfNeeded() {
        if (eventQueue_.size() >= EVENT_QUEUE_SIZE_THRESHOLD) {
            connectionQueue_.recordEvents(eventQueue_.events());
        }
    }
```

#####4.2.2.10 ConnectionQueue.java
需要被发送的各种数据，包括前面说过的event和crash等，都在此提供发送接口，实际发送的时候都会被转换为connection,持久化添加到connectionQueue中，Tick的时候由ConnectionProcessor从Store中取出发送到服务端。

```
//该方法会被定时触发
 void recordEvents(final String events) {
        checkInternalState();
        final String data = "app_key=" + appKey_
                          + "&timestamp=" + Countly.currentTimestamp()
                          + "&hour=" + Countly.currentHour()
                          + "&dow=" + Countly.currentDayOfWeek()
                          + "&events=" + events;

        store_.addConnection(data);

        tick();
    }
```
tick与connectionProcessorFuture_要解释下

```
void tick() {
        if (!store_.isEmptyConnections() && (connectionProcessorFuture_ == null || connectionProcessorFuture_.isDone())) {
            ensureExecutor();
            connectionProcessorFuture_ = executor_.submit(new ConnectionProcessor(serverURL_, store_, deviceId_, sslContext_));
        }
    }
```
其他数据也跟Events类似，包括session,location,userData,CrashReport

```
void beginSession()
void updateSession(final int duration)
void endSession(final int duration)

void sendCrashReport(String error, boolean nonfatal)
void sendUserData()
```
#####4.2.2.11 ConnectionProcessor.java
是个Runnable，每次被Run的时候，从Store中取出当前所有的Connections，用http发送。

```
URLConnection urlConnectionForEventData(final String eventData)
```
拼装url并打开Conn

```
public void run()
```
从Store中取出数据，调用urlConnectionForEventData（）生成conn,发起请求。


###6. 修改完善  
在完成了上面 5 个部分后，移动模块顺序，将  
`2. 详细设计` -> `2.1 核心类功能介绍` -> `2.2 类关系图` -> `3. 流程图` -> `4. 总体设计`  
顺序变为  
`2. 总体设计` -> `3. 流程图` -> `4. 详细设计` -> `4.1 类关系图` -> `4.2 核心类功能介绍`  
并自行校验优化一遍，确认无误后将文章开头的  
`分析状态：未完成`  
变为：  
`分析状态：已完成`  

本期校对会由专门的`Buddy`完成，可能会对分析文档进行一些修改，请大家理解。  

**完成时间**  
- `两天内`完成  

**到此便大功告成，恭喜大家^_^**  