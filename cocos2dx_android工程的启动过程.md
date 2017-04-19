##Cocos2dx Android工程的启动过程

###1、安卓工程下的设置启动activity为src下面的AppActivity,启动调用的onCreate并没有做过多的事情，只是调用了父类Cocos2dxActivity的onCreate。AppActivity代码如下：
```	
import org.cocos2dx.lib.Cocos2dxActivity;

public class AppActivity extends Cocos2dxActivity {
	@Override
	protected void onCreate(final Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		......
    }
}
```

###2、Cocos2dxActivity在cocos/platform/android/java/src/org/cocos2dx/lib/Cocos2dxActivity.java里，查看onCreate,代码如下： 
```
@Override
	protected void onCreate(final Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		initFMOD(); //加载声音库
		try {
			Class.forName("android.os.AsyncTask");
		} catch (Throwable ignor) {
			ignor.printStackTrace();
		}
		sContext = this;
		PSNetwork.init(sContext); //初始化安卓网络连接服务
		if (Build.VERSION.SDK_INT >= 23) {
			requestUserPermissions(); //系统的部分授权
		}
		mMacAddress = MacAddressUtil.getMacAddress(sContext);//获取wifi的MAC地址
		CocosPlayClient.init(this, false); //暂时无用
		boolean isLoadOK = onLoadNativeLibraries(); //把工程中libs下面的so文件load进来，定义在AndroidManifest, meta-data标签下，android.app.lib_name. 最终在包的data/data/com.XXX.XXX/lib下面
		if (false == isLoadOK) {
			return;
		}

		this.mHandler = new Cocos2dxHandler(this);//处理安卓的弹窗等

		Cocos2dxHelper.init(this);

		this.mGLContextAttrs = getGLContextAttrs();//获取OpenGLEs的相关属性
		this.init(); //说明如下文

		if (mVideoHelper == null) {
			mVideoHelper = new Cocos2dxVideoHelper(this, mFrameLayout);
		}

		if (mWebViewHelper == null) {
			mWebViewHelper = new Cocos2dxWebViewHelper(mFrameLayout);
		}

        if(mEditBoxHelper == null){
            mEditBoxHelper = new Cocos2dxEditBoxHelper(mFrameLayout);
        }
		if (null == mScreenListener) {
			mScreenListener = new ScreenListener(this);
			mScreenListener.begin(this);
		}
	}
```

###3、Cocos2dxActivity的init函数如下：
```
	public void init() {
		// FrameLayout
		ViewGroup.LayoutParams framelayout_params = new ViewGroup.LayoutParams(
				ViewGroup.LayoutParams.MATCH_PARENT,
				ViewGroup.LayoutParams.MATCH_PARENT);
		mFrameLayout = new ResizeLayout(this); //继承自FrameLayout,看成是一块画布(canvas),其他控件添加在上面
		mFrameLayout.setLayoutParams(framelayout_params);

		// Cocos2dxEditText layout
		ViewGroup.LayoutParams edittext_layout_params = new ViewGroup.LayoutParams(
				ViewGroup.LayoutParams.MATCH_PARENT,
				ViewGroup.LayoutParams.WRAP_CONTENT);
		Cocos2dxEditBox edittext = new Cocos2dxEditBox(this);
		edittext.setLayoutParams(edittext_layout_params);//输入框

		// ...add to FrameLayout
		mFrameLayout.addView(edittext);

		// Cocos2dxGLSurfaceView
		this.mGLSurfaceView = this.onCreateView();//创建游戏的渲染，接受输入事件的OpenGL类

		// ...add to FrameLayout
		mFrameLayout.addView(this.mGLSurfaceView);//添加到画布上

		// Switch to supported OpenGL (ARGB888) mode on emulator
		if (isAndroidEmulator())
			this.mGLSurfaceView.setEGLConfigChooser(8, 8, 8, 8, 16, 0);

		this.mGLSurfaceView.setCocos2dxRenderer(new Cocos2dxRenderer());//注册自主实现的渲染器,内容如下
		this.mGLSurfaceView.setCocos2dxEditText(edittext);//输入框

		this.onCreateFrameLayout();
		// Set framelayout as the content view
		setContentView(mFrameLayout);//设置这个Activity的显示界面
	}


```

###4、Cocos2dxRenderer,cocos2dx的渲染器，继承自android.opengl.GLSurfaceView.Renderer，当3中的GLSurfaceView被创建的时候会调用render的onSurfaceCreated()方法;  当GLSurfaceView大小或者横竖屏发生变化的时候调用render的onSurfaceChanged()方法； 当系统每一次重新画GLSurfaceView的时候，调用onDrawFrame()方法。所以Cocos2dxRender对这三个方法进行了重写。
```
	@Override
	public void onSurfaceCreated(final GL10 GL10, final EGLConfig EGLConfig) {
		Cocos2dxRenderer.nativeInit(this.mScreenWidth, this.mScreenHeight);//此处调用了一个定义为native的函数，设置glview等，通过jni来访问c++实现的方法，接口实现在cocos/platform/android/javaactivity-android.cpp里面，下面的5会继续讲
		this.mLastTickInNanoSeconds = System.nanoTime();
		mNativeInitCompleted = true;
		try{
			Cocos2dxActivity activity = (Cocos2dxActivity) Cocos2dxActivity.getContext();
			activity.onSurfaceCreated(this, GL10, EGLConfig);
		}catch( Throwable e ){
			e.printStackTrace();
		}
	}

```

```
@Override
	public void onDrawFrame(final GL10 gl) { //系统自动每秒钟调用60次这个函数
		/*
		 * No need to use algorithm in default(60 FPS) situation, since
		 * onDrawFrame() was called by system 60 times per second by default.
		 */
		if( true == mIsPaused ){
			return;
		}
		if( mDelayResumeCount <= DELAY_RESUME_COUNT ){
			mDelayResumeCount = mDelayResumeCount + 1;
			if( mDelayResumeCount == DELAY_RESUME_COUNT ){
				Cocos2dxRenderer.nativeOnResume();
				try{
					Cocos2dxActivity activity = (Cocos2dxActivity) Cocos2dxActivity.getContext();
					activity.nativeResume();
				}catch( Throwable e ){
					e.printStackTrace();
				}
			}
			return;
		}
		if (sAnimationInterval <= 1.0 / 60 * Cocos2dxRenderer.NANOSECONDSPERSECOND) {
			Cocos2dxRenderer.nativeRender(); //大于等于每秒60帧则不经过算法处理，直接执行nativeRender，将在6中有说明
		} else {
			final long now = System.nanoTime();
			final long interval = now - this.mLastTickInNanoSeconds;

			if (interval < Cocos2dxRenderer.sAnimationInterval) { //按照设置的帧数，如果没有到时间，则sleep到相应的时间
				try {
					Thread.sleep((Cocos2dxRenderer.sAnimationInterval - interval)
							/ Cocos2dxRenderer.NANOSECONDSPERMICROSECOND);
				} catch (final Exception e) {
				}
			}
			/*
			 * Render time MUST be counted in, or the FPS will slower than
			 * appointed.
			 */
			this.mLastTickInNanoSeconds = System.nanoTime();
			Cocos2dxRenderer.nativeRender();
		}
		try{
			Cocos2dxActivity activity = (Cocos2dxActivity) Cocos2dxActivity.getContext();
			activity.onDrawFrame(this, gl);
		}catch( Throwable e ){
			e.printStackTrace();
		}
	}

```

```
	@Override
	public void onSurfaceChanged(final GL10 GL10, final int width,
			final int height) {
		Cocos2dxRenderer.nativeOnSurfaceChanged(width, height);
		try{
			Cocos2dxActivity activity = (Cocos2dxActivity) Cocos2dxActivity.getContext();
			activity.onSurfaceChanged(this, GL10, width, height);
		}catch( Throwable e ){
			e.printStackTrace();
		}
	}
```

###5、 4里面onSurfaceCreated的ativeInit的实现放在cocos/platform/android/javaactivity-android.cpp，方法如下：
```
void Java_org_cocos2dx_lib_Cocos2dxRenderer_nativeInit(JNIEnv*  env, jobject thiz, jint w, jint h)
{
    auto director = cocos2d::Director::getInstance();
    auto glview = director->getOpenGLView();
    if (!glview)
    {
        glview = cocos2d::GLViewImpl::create("Android app");
        glview->setFrameSize(w, h);
        director->setOpenGLView(glview); //设置glview

        //cocos_android_app_init(env, thiz);

        cocos2d::Application::getInstance()->run(); //程序开始运行，android的Application实现放在CCApplication-android.cpp,run的代码如下
    }
    else
    {
        cocos2d::GL::invalidateStateCache();
        cocos2d::GLProgramCache::getInstance()->reloadDefaultGLPrograms();
        cocos2d::DrawPrimitives::init();
        cocos2d::VolatileTextureMgr::reloadAllTextures();

        cocos2d::EventCustom recreatedEvent(EVENT_RENDERER_RECREATED);
        director->getEventDispatcher()->dispatchEvent(&recreatedEvent);
        director->setGLDefaultValues();
    }
}

```

```
int Application::run()
{
    // Initialize instance and cocos2d.
    if (! applicationDidFinishLaunching()) //applicationDidFinishLanunching在自己的Classes/AppDelegate进行重写,游戏已经启动
    {
        return 0;
    }
    
    return -1;
}
```

```
bool AppDelegate::applicationDidDinishLaunching()
{
	......
	director->setOpenGLView(glview);
	director->setAnimationInterval(1/30.f); //设置帧数，会调用Application-android的setAnimationInterval,再通过JniHelper调用Cocos2dxRenderer中的setAnimationInterval
	director->runWithScene(scene); //
	return true
}
```

###6、关于4中onDrawFrame涉及到的函数nativeRender，它也是一个native类型的函数，实现放在cocos/platform/android/jni/Java_org_cocos2dx_lib_Cocos2dxRenderer.cpp
```
    JNIEXPORT void JNICALL Java_org_cocos2dx_lib_Cocos2dxRenderer_nativeRender(JNIEnv* env) {
        cocos2d::Director::getInstance()->mainLoop(); //进入游戏的主循环，Director的mainLoop，事件的分发，渲染,内存池的管理
    }
```