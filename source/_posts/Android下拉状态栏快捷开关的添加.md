---
title: Android下拉状态栏快捷开关的添加
date: 2020-03-18 22:55:00
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---
[下拉通知栏快捷开关的添加](https://blog.csdn.net/lyjit/article/details/51579067)  
### 一：快捷开关功能以及开关状态的实现

#### 第一步： 
首先看一下原生的下拉菜单按键的布局在哪里。打开这个路径的文件：frameworks/base/packages/SystemUI/res/values/config.xml


```
<string name="quick_settings_tiles_default" translatable="false">
   wifi,bt,inversion,cell,airplane,rotation,flashlight,settings,dataconnection,location,screenshot,cast,hotspot,hotknot,audioprofile
</string>
```

对！自适应的布局，这些快捷按键的定义都在这里。要是想换成自己定义的按键开关，只要在这里替换就行，但这只是第一步。  
我这里暂且换成自己定义的布局：wifi,蓝牙,无线麦克,FM

```
<string name="quick_settings_tiles_default" translatable="false">
        wifi,bluetooth,wirelessmic,fm
</string>
```

#### 第二步：
在config.xml文件中定义我们想要的开关名称之后，还需要打开如下路径来添加相关的内容：  
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/QSTileHost.java
在如下方法里面添加：
  
```
/**
     * 获取到预定义好的各快捷图标的QSTile
     * */
    private QSTile<?> createTile(String tileSpec) {
        IQuickSettingsPlugin quickSettingsPlugin = PluginFactory
                .getQuickSettingsPlugin(mContext);
        if (tileSpec.equals("wifi")) return new WifiTile(this);
        //add by lyj
        else if (tileSpec.equals("bluetooth")) return new CallBluetoothTile(this);
        else if (tileSpec.equals("wirelessmic")) return new WirelessMicTile(this);
        else if (tileSpec.equals("fm")) return new FMTile(this); //要添加的FM
 
        ......
 
	}
```

这里主要是获取到预定义好的各个快捷图标的QSTile。其中FMTile文件是什么的，接着往下看
#### 第三步：
这一步很重要，首先看一下这个路径：frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tiles  
这个路径下面就是你定义的开关实现功能的地方，这里新添加的FM在原有的快捷开关里面是没有的，所以要在这里新添加一个文件，咱们这里暂且起名为FMTile.java，也就是之前所说的FMTile文件。  
新建好类之后要继承 QSTile<QSTile.BooleanState>  
这样会生成一些方法，下面就来看一下这个文件各个方法的作用：


```
public class FMTile extends QSTile<QSTile.BooleanState>{
	
	public FMTile(com.android.systemui.qs.QSTile.Host host) {
		super(host);
		// TODO Auto-generated constructor stub
	}
 
	@Override
	public void setListening(boolean listening) {
		// TODO Auto-generated method stub
	}
 
	@Override
	protected com.android.systemui.qs.QSTile.BooleanState newTileState() {
		// TODO Auto-generated method stub
		return new BooleanState();
	}
	public void setFmModeUpdate(){
		handleRefreshState(null);
        }
	@Override
       protected String getQSTileViewMarkBit(){
	 	return "fm" ;
	 }
 
	 @Override
	protected void handleClick() {
		// TODO Auto-generated method stub
                //开关默认关闭
                int mState = Settings.System.getInt(mContext.getContentResolver(), Settings.System.FM_SYSTEMUI, 1);
		boolean mCloseState = (mState == 0);
		Settings.System.putInt(mContext.getContentResolver(),Settings.System.FM_SYSTEMUI, mCloseState ? 1 : 0);
		
	}
 
	@Override
	protected void handleUpdateState(
			com.android.systemui.qs.QSTile.BooleanState state, Object arg) {
		
		int mState = Settings.System.getInt(mContext.getContentResolver(), Settings.System.FM_SYSTEMUI, 1);
		boolean mCloseState = (mState == 0);
		state.visible = true ;
		state.label = mContext.getString(R.string.quick_settings_fm_title);
		state.icon = ResourceIcon.get(mCloseState ? R.drawable.ic_settings_fm_on : R.drawable.ic_settings_fm_off);//图片状态
		if(mCloseState){//open
			//开关打开相关功能的操作
			
		}else{
			//开关关闭相关功能的操作
			
		}
	}
 
}
```

其中Settings.System.FM_SYSTEMUI 是我们在settingProvader中定义的值，主要是控制与外界开关同步的问题，这个我们后面会有详细的介绍，这些方法中最重要的是handleClick()和handleUpdateState(...),每次点击开关会先走handleclick()方法，然后去刷新页面，刷新页面的处理就是在handleUpdateState(..)中处理。  
这个方法里面有图片状态的改变，以及功能的实现。这个时候开关默认的是关闭，要是默认打开的话只要在两个方法中把mstate = Settings.System.getInt(*, *, 1);  把1改为0即可。  
这里还要注意的方法首先是FMTile(..)这个方法是初始化用的，也就是开机之后只走一次，也就相当我Activity里面的Oncreate()方法,如果你有什么要初始化的东西可以在这里面定义。还有特别注意的两个方法setFmModeUpdate()和getQSTileViewMarkBit()。这两个方法不是文件自动生成的，需要你手动添加，那么这两个方法有什么作用呢，其实这两个方法也很重要，要是没有这两个方法的话点击开关是没有作用的，也就是没有起到刷新页面的作用，那这两个方法在那实现的呢，来接着看下面的解释：
#### 第四步：

结合第三步最后的疑问我们来看一下这个文件：frameworks/base/packages/SystemUI/src/com/android/systemui/qs/QSTile.java

```
protected QSTile(Host host) {
        mHost = host;
        mContext = host.getContext();
        mHandler = new H(host.getLooper());
        setChangeObserver();
    }
    
    public void setChangeObserver(){ //实时监听状态的变化
    	mContext.getContentResolver().registerContentObserver(
                Settings.System.getUriFor(Settings.System.FM_SYSTEMUI),
                true, mFmModeChangeObserver);
	.....
		
    }
    private ContentObserver mFmModeChangeObserver = new ContentObserver(new Handler()) {
		@Override
		public void onChange(boolean selfChange) {
			setFmModeUpdate();
		}
	};
	
    public void setFmModeUpdate(){
 
    }
```

    这些添加之后你的开关状态就能其作用了，但到这里如果你编译工程的时候会发现编译报错，如下图：


这是怎么回事呢，这是因为你添加的FMTile.java文件里面的方法是从一个超类型实现的。

这个时候你还需有在QSTile.java文件中添加getQSTileViewMarkBit()方法，如下：


```
//
    protected QSTileView mQSTileView ;
    public QSTileView createTileView(Context context) {
    	
    	mQSTileView = new QSTileView(context) ;
    	mQSTileView.setMarkBit(getQSTileViewMarkBit());//add 
    	Log.i("lyj_create", "mQSTileView = "+mQSTileView);
        return mQSTileView;
    }
    protected String getQSTileViewMarkBit(){
		return null ;
    }
```

到这里FM开关的添加就结束了。同样的方法添加其它快捷按键开关原理都是一样的。
###   二：下拉FM开关与外界开关同步

   其实刚才我们已经有所了解了，主要就是Settings值的作用，但这里面还涉及到一个重要的知识点，先不着急，我们先打开FM的应用代码，看怎么实现同步更新：

   首先在你应用里面控制开关的地方加上Settings.System.putInt(MainActivity.this.getContentResolver(), Settings.System.FM_SYSTEMUI, 0);  //0代表打开 关闭的时候调用Settings.System.putInt(MainActivity.this.getContentResolver(), Settings.System.FM_SYSTEMUI, 1);  //1代表关闭

   关键的代码来了：也就是我们之前说的实现同步更新的 ContentObserver 内容观察者

首先看一下代码：

	
```
private void registerContentObserver() {
		this.getContentResolver().registerContentObserver(
				Settings.System.getUriFor(Settings.System.FM_SYSTEMU), true, mFmcContentObserver);
	}
 
	private void unregisterContentObserver() {
		this.getContentResolver().unregisterContentObserver(mFmcContentObserver);
	}
 
	private ContentObserver mFmcContentObserver=new ContentObserver(new Handler()) {
 
		@Override
		public void onChange(boolean selfChange) {
			int mState = Settings.System.getInt(getActivity().getContentResolver(), Settings.System.FM_SYSTEMU, 0);
			boolean mCloseState = (mState == 0);
			if(mCloseState){//打开的相关操作
				mSwitchPreference.setChecked(true);
				Log.i("lyj_wire","mikeState: = "+mikeState);
			}else{//关闭的相关操作
				mSwitchPreference.setChecked(false);
				Log.i("lyj_wire","11mikeState:"+mikeState);
			}
			
		}
		
	};
```

在onChange方法里面主要是观察Settings值的变化，然后根据值的变化控制你应用的开关。我们定义的这个值在状态栏快捷开关里面也有所控制，这个刚才已经介绍。那么这个时候你会问，我们在应用里面添加观察者可是没有在下拉状态栏里添加，这样能同步吗？其实SystemUi里面已经添加过了,你只需要在你的应用里添加内容观察者即可。注意在初始化的地方要注册registerContentObserver() ，在退出的时候调用unregisterContentObserver()方法即可。这样以来就能实现应用开关和状态栏下拉快捷开关的同步。
这里还涉及到一个知识点：就是打开开关，不管是在你的应用里还是在状态栏快捷开关里，打开FM后在状态栏顶部需要有一个图标显示你打开了FM，关闭时图片消失。这个功能的实现我会在后续的文章中介绍。这里就不做说明了。

  相信看到这里你会对SystemUi状态栏的开发有新的帮助。