countly-sdk-android 源码解析
====================================
>  项目地址：[countly-sdk-android](https://github.com/Countly/countly-sdk-android)，分析的版本：[16.02.01 Release](https://github.com/Countly/countly-sdk-android/commit/c105d06e573703d9e29d5c92068a41befa36f43f)，Demo 地址：[countly-sdk-android-demo](https://github.com/Labmem003/zhenzhi-open-project-analysis/tree/master/countly-sdk-android-demo)    
 分析者：[振之](https://github.com/Labmem003)，分析状态：未完成


###1. 功能介绍  
####1.1 Countly
Countly 是一款类似于友盟的移动&Web应用通用的实时统计分析系统，专注于易用性、扩展性和功能丰富程度。不同之处是Countly是开源的，任何人都可以将Countly客户端部署在自己的服务器并将开发工具包整合到他们的应用程序中。比友盟要简洁干净，而且这下数据和程序都完全处于自己控制之下了。

这里分析的是其Android端的sdk, 以了解和学习移动应用统计类的工具收集App的使用情况和终端用户的行为的机制。主要功能包括App基本数据收集、自定义事件记录、崩溃报告。
####1.2 基本使用
待补充。。。

###2. 详细设计
###2.1 类详细介绍
结构很简单，一共就两个包，countly核心包和openudid包。
上图。
countly包解决统计什么，怎么实施统计；而openudid包解决如何标记统计的数据来自何方。

####2.1.1 openudid包
先来看看比较简单的openudid包，她是一个设备标识方案，能提供一个设备通用统一标识符（Unique Device IDentifier/UDID）。如果同一台设备上有多个App都用了这个包来生成UDID，他们获取的UDID是一致的（即所谓设备标识）。当我们将统计数据发送给服务端时，会将UDID附上。不难想到，之后服务端算日活、描述用户特征、事件追踪等等各种后续的数据分析肯定都离不开UDID，算是很必要的基础设施。实际上她也是一个开源包[OpenUDID](https://github.com/vieux/OpenUDID)。
#####2.1.1.1 OpenUDID_service.java
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
#####2.1.1.2 OpenUDID_manager.java
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

####2.1.2 countly包
#####2.2.1.1 OpenUDIDAdapter.java
包装了UDID包，提供sync（），getOpenUDID（）。但是是用动态反射的方法封装的，不明白为什么。官方的commit 木essage说了一句：call OpenUDID dynamically so that including the OpenUDID source is not necessary to get the Countly Android SDK to work when an app provides it's own deviceID。
看懂的请告诉我。
#####2.2.1.1 DeviceId.java
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
#####2.2.1.1 DeviceInfo.java
一个纯POJO类，用来存放设备信息，如设备名称、设备分辨率、版本号等。

主要方法：
static String getMetrics(final Context context)，返回url-encoded的属性json字符串。
#####2.2.1.1 CrashDetails.java
#####2.2.1.1 Event.java
#####2.2.1.1 EventQueue.java
#####2.2.1.1 CountlyStore.java
#####2.2.1.1 UserData.java
#####2.2.1.1 Countly.java
#####2.2.1.1 ConnectionQueue.java
#####2.2.1.1 ConnectionProcessor.java

###2.2 类关系图
类关系图，类的继承、组合关系图，可是用 [StarUML](http://staruml.io/) 工具。  

**完成时间**  
- 根据项目大小而定，目前简单根据项目 Java 文件数判断，完成时间大致为：`文件数 * 7 / 10`天，特殊项目具体对待  

###3. 流程图
主要功能流程图  
- 如 Retrofit、Volley 的请求处理流程，Android-Universal-Image-Loader 的图片处理流程图  
- 可使用 [Google Drawing](https://docs.google.com/drawings)、[Visio](http://products.office.com/en-us/visio/flowchart-software)、[StarUML](http://staruml.io/) 等工具完成，其他工具推荐？？  
- 非所有项目必须，不需要的请先在群里反馈  

**完成时间**  
- `两天内`完成  

###4. 总体设计
整个库分为哪些模块及模块之间的调用关系。  
- 如大多数图片缓存会分为 Loader 和 Processer 等模块。  
- 可使用 [Google Drawing](https://docs.google.com/drawings)、[Visio](http://products.office.com/en-us/visio/flowchart-software)、[StarUML](http://staruml.io/) 等工具完成，其他工具推荐？？  
- 非所有项目必须，不需要的请先在群里反馈。  

**完成时间**  
- `两天内`完成  

###5. 杂谈
该项目存在的问题、可优化点及类似功能项目对比等，非所有项目必须。  

**完成时间**  
- `两天内`完成  

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