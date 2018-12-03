title: LeakCanary分析详解
date: 2015-12-24 19:11:12
tags:
	- android
categories: tech
comments: true
---
## 引言
内存泄露是Android开发过程中非常常见的问题，指的是进程中某些已经完成使命的垃圾对象始终占据着内存空间，直接或间接保持对GCROOTS的引用，导致无法被GC回收。对于Android系统而言，因为Android每个进程有自己的内存上限，当app使用内存超过可申请的内存时，就会出现Out Of Memory的错误。如果应用存在内存泄露，出现OOM，很难从堆栈中直接分析出问题原因。
对于Android应用内存泄露的检测，通常处理的情况是：

1. 通过统计平台发现相关的OOM问题上报；
2. 重现内存泄露问题**（本步骤最需要花大量的时间和人力）**，并记录堆栈信息并dump内存得到相关的hprof文件；
3. 用MAT等工具的方式来分析查找到泄露对象以及到GC ROOTS的最短强引用路径；
4. 修复问题。

内存泄露的原因可能有很多种情况，比如非静态内部类的静态实例、内部类handler消息传递、注册某个对象后未反注册、集合中对象没及时清理、资源对象未关闭、图片读取、Adapter未缓存View等。对此按照上述的人工排查的方式来处理往往需要的时间和人力很大，本文主要是介绍Square公司推出的LeakCanary,介绍其使用和相关原理。

## LeakCanary的使用

LeakCanary的源工程地址为<https://github.com/square/leakcanary>。
### 1.接入项目依赖
* 在Android Studio中使用（**添加Gradle依赖**）
	
	通过debugCompile和releaseCompile来控制debug和release版本，release版本大家肯定不希望还带有自动内存监测的入口。
	
	
		dependencies {
		debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3.1'
		releaseCompile ‘com.squareup.leakcanary:leakcanary-android-no-op:1.3.1'
		}
* 在Eclipse 中使用（**作为Library依赖**）
	
	LeakCanary项目默认没有提供Eclipse的依赖方式，相关开源工程地址为<https://github.com/teffy/LeakcanarySample-Eclipse>。

 		android.library.reference.1=../leakcanarylib 
 	**PS：**对于某些特殊的工程，不便于添加依赖工程并且也不是gradle构建的项目，可以将相关的java文件和res文件拷贝到对应的资源目录，并且在AndroidManifest添加下列代码也可以接入LeakCanary。
 		
 		<service android:name="com.squareup.leakcanary.internal.HeapAnalyzerService"
            android:enabled="false"
            android:process=":leakcanary" />
        <service android:name="com.squareup.leakcanary.DisplayLeakService"
            android:enabled="false" />
        <activity
            android:name="com.squareup.leakcanary.internal.DisplayLeakActivity"
            android:enabled="false"
            android:icon="@drawable/__leak_canary_icon"
            android:label="@string/__leak_canary_display_activity_label"
            android:taskAffinity="com.squareup.leakcanary"
            android:theme="@style/__LeakCanary.Base" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

### 2. 加入检测代码
* 监听可能泄露的对象(**添加RefWatcher来watch对象**)
		
		RefWatcher refWatcher = {...};
		refWatcher.watch(badObject);//监听本该被gc的对象
		
* 对于Android 4.0以上版本的Activity监听(**LeakCanary.install(Application)**)

		public class ExampleApplication extends Application {
			public static RefWatcher getRefWatcher(Context context) {
				ExampleApplication application = (ExampleApplication)context.getApplicationContext();
				return application.refWatcher;
			}
			private RefWatcher refWatcher;
			@Override public void onCreate() {
				super.onCreate();
				refWatcher = LeakCanary.install(this);
			}
		}
* 对于Fragment和Android 4.0以下的Activity监听(**基类OnDestory加入watch**)
		
		//在Activity和Fragment的基类的onDestory中添加watch
		public abstract class BaseFragment extends Fragment {
			@Override public void onDestroy() {
			super.onDestroy();
				RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
				refWatcher.watch(this);
			}
		}
		
### 3. 一般的输出结果

![输出结果](leaktrace.png "输出结果")

当发生泄漏时，系统通知栏会弹窗如图左上角所示。并且会在应用中出现一个leaks的入口，点击进去会出现泄露路径相关描述。通过右上角的button可以分享相关的leakInfo以及对应的dump出来的hprof文件。对于hprof文件，可以通过分析其中的KeyedWeakReference的引用（稍后会在原理处说明做法的原因），即可得到内存泄露发生时检测的对象，能够比较快的定位问题。
	
### 4. 自定义泄露处理流程
	public class LeakUploadService extends DisplayLeakService {
		@Override
		protected void afterDefaultHandling(HeapDump heapDump,AnalysisResult result, String leakInfo) {
			if (!result.leakFound || result.excludedLeak) {
		      return;
		    } 
		    //可以做相关处理，比如上传云服务器
		    }
	    }
	    public class ExampleApplication extends Application
	    {
		    protected RefWatcher installLeakCanary() {
		    return LeakCanary.install(app, LeakUploadService.class);
	    }
    }
   
### 5. 自定义忽略的泄露
* 忽略泄露的类
		
		//忽略com.example.Exampleclass的exampleField的泄露
		public class ExampleApplication extends Application {
			protected RefWatcher installLeakCanary() {
				ExcludedRefs excludedRefs = AndroidExcludedRefs.createAppDefaults()
				.instanceField("com.example.ExampleClass", "exampleField")
				.build();
				return LeakCanary.install(this, DisplayLeakService.class, excludedRefs);
			}
		}
        
* 忽略需要监听的Activity
		
		//不用监听NoLeakActivity
		Public class ExampleApplication extends Application {
			protected RefWatcher installLeakCanary() {
				final RefWatcher refWatcher = androidWatcher(application);
				registerActivityLifecycleCallbacks(new ActivityLifecycleCallbacks() {
		        public void onActivityDestroyed(Activity activity) {
		          if (activity instanceof  NoLeakActivity) {
		              return;
		          }
		          refWatcher.watch(activity);
		        }
		        // ...
		        });
	        	return refWatcher;
	        }
        }

## LeakCanary自动检测原理
* LeakCanary.install(application)通过registerActivityLifecycleCallbacks在Activity的onDestory中加入RefWatcher监听Activity。
* RefWatcher.watch() 创建一个 KeyedWeakReference 到要被监控的对象。
* 然后在后台线程检查引用是否被清除，如果没有，调用GC。如果引用还是未被清除，把 heap 内存 dump 到 APP 对应的文件系统中的一个 .hprof 文件中。
* 在另外一个进程中的 HeapAnalyzerService 有一个 HeapAnalyzer解析这个文件。
* 得益于唯一的 reference key, HeapAnalyzer 找到 KeyedWeakReference，定位内存泄露。
* HeapAnalyzer 计算 到 GC roots 的最短强引用路径，并确定是否是泄露(如果是已知系统级泄漏，会自动忽略掉)。如果是的话，建立导致泄露的引用链。（过滤已知泄漏）
* 引用链传递到 APP 进程中的 DisplayLeakService， 并以通知的形式展示出来或者自行处理结果。


## LeakCanary代码结构
	--leakcanary     
        --leakcanary-analyzer         hprof文件分析模块
              --HeapAnalyzer          利用HAHA和ShortestPathFinder分析内存泄露是否误报
              --ShortestPathFinder    查找泄露对象到GC ROOTS的最短强引用路径
        --leakcanary-android          Android使用模块
              --LeakCanary            入口类
              --ActivityRefWatcher    对于4.0以上监听Activity Destroy，加入监听
              --AndroidExcludedRefs   判断泄露是否应该忽略
              --AndroidWatchExecutor  在主线程空闲时，将检测任务抛入Thread处理
              --HeapAnalyzerService   分析hprof 
              --DisplayLeakService    展示分析结果
              --DisplayLeakActivity   分析结果显示界面
         --leakcanary-watcher         检测模块
              --RefWatcher            通过弱引用队列分析是否泄露，二次确认
              --GcTrigger             若发现没有释放，触发gc
        
## LeackCanary源码分析
LeakCanary的整个库相对比较复杂，包含了泄漏的监测和相关泄漏dump出来的的hprof分析。源码分析主要包含六个主要的关键源码：

* [ActivityRefWatcher](https://github.com/square/leakcanary/blob/master/leakcanary-android/src/main/java/com/squareup/leakcanary/ActivityRefWatcher.java)

	通过registerActivityLifecycleCallbacks在Activity的onDestory中加入监听流程（只支持Android 4.0以上）。
		
		private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
		new Application.ActivityLifecycleCallbacks() {
			@Override public void onActivityDestroyed(Activity activity) {
				refWatcher.watch(activity);
			}
		};

* [RefWatcher](https://github.com/square/leakcanary/blob/master/leakcanary-watcher/src/main/java/com/squareup/leakcanary/RefWatcher.java)

	创建KeyedWeakReference并通过AndroidWatchExecutor（主要是为了确保在主线程空闲的时候进行检测）执行监听流程。
	
		public void watch(Object watchedReference, String referenceName) {
		    checkNotNull(watchedReference, "watchedReference");
		    checkNotNull(referenceName, "referenceName");
		    if (debuggerControl.isDebuggerAttached()) {
		      return;
		    }
		    final long watchStartNanoTime = System.nanoTime();
		    String key = UUID.randomUUID().toString();
		    retainedKeys.add(key);
		    final KeyedWeakReference reference =
		        new KeyedWeakReference(watchedReference, key, referenceName, queue);
		    watchExecutor.execute(new Runnable() {
		      @Override public void run() {
		        ensureGone(reference, watchStartNanoTime);
		      }
		    });
  		}
  判断对象是否引用消除主要是通过判断retainedKeys中是否存在对应KeyedWeakReference的key。
  监听流程主要是先移除回收队列中存在的相关的key，确认是否引用消除，没有则进行`gcTrigger.runGc()`来gc，再次移除回收队列中存在的相关的key，如果引用仍未清除，则判断内存泄漏。
	  
		  void ensureGone(KeyedWeakReference reference, long watchStartNanoTime) {
		    long gcStartNanoTime = System.nanoTime();
		    removeWeaklyReachableReferences();
		    if (gone(reference) || debuggerControl.isDebuggerAttached()) {
		      return;
		    }
		    gcTrigger.runGc();
		    removeWeaklyReachableReferences();
		    if (!gone(reference)) {
		      //dump内存并作相关处理
		    }
		  }
		
		  private void removeWeaklyReachableReferences() {
		    KeyedWeakReference ref;
		    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
		      retainedKeys.remove(ref.key);
		    }
		  }
  
* [AndroidWatchExecutor](https://github.com/square/leakcanary/blob/master/leakcanary-android/src/main/java/com/squareup/leakcanary/AndroidWatchExecutor.java)
	
	AndroidWatchExecutor是为了保证监听流程的执行是在主线程空闲的时候进行检测。
	
		@Override public void execute(final Runnable command) {
		    if (isOnMainThread()) {
		      executeDelayedAfterIdleUnsafe(command);
		    } else {
		      mainHandler.post(new Runnable() {
		        @Override public void run() {
		          executeDelayedAfterIdleUnsafe(command);
		        }
		      });
		    }
		  }
		  
		  private boolean isOnMainThread() {
		    return Looper.getMainLooper().getThread() == Thread.currentThread();
		  }
		
		  private void executeDelayedAfterIdleUnsafe(final Runnable runnable) {
		    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
		      @Override public boolean queueIdle() {
		        backgroundHandler.postDelayed(runnable, 5000);
		        return false;
		      }
		    });
		  }
	
	保证在主线程空闲的时候检测的主要原因是因为leakcanary检测过程中一旦发现疑似泄漏就会dumphprof并进行分析，虽然dump的过程在后台线程，但是dumphprof一定会执行到对应的dump.cc，执行dumpHeap时所有的线程都会暂停，会造成突然卡顿；所以需要在主线程空闲的时候才进行检测。
	
		void DumpHeap(const char* filename, int fd, bool direct_to_ddms) {
		    CHECK(filename != NULL);
		    Runtime::Current()->GetThreadList()->SuspendAll();
		    Hprof hprof(filename, fd, direct_to_ddms);
		    hprof.Dump();
		    Runtime::Current()->GetThreadList()->ResumeAll();
		}


* [GcTrigger](https://github.com/square/leakcanary/blob/master/leakcanary-watcher/src/main/java/com/squareup/leakcanary/GcTrigger.java)
	
	GCTrigger主要是执行GC并且wait 100毫秒，这个设定参考自[FinalizationTest](https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/java/lang/ref/FinalizationTester.java "FinalizationTest")。
	
		  GcTrigger DEFAULT = new GcTrigger() {
		    @Override public void runGc() {
		      Runtime.getRuntime().gc();
		      enqueueReferences();
		      System.runFinalization();
		    }
		
		    private void enqueueReferences() {
		      try {
		        Thread.sleep(100);
		      } catch (InterruptedException e) {
		        throw new AssertionError();
		      }
		    }
		  };
	其中使用Runtime.getRuntime().gc()比System.gc()更能保证GC执行，在Android 5.0以下的源码中，System.gc()
			
		public static void gc() {
	        Runtime.getRuntime().gc();
	    }
	Android 5.0以上系统中：
		
		public static void gc() {
	        boolean shouldRunGC;
	        synchronized(lock) {
	            shouldRunGC = justRanFinalization;
	            if (shouldRunGC) {
	                justRanFinalization = false;
	            } else {
	                runGC = true;
	            }
	        }
	        if (shouldRunGC) {
	            Runtime.getRuntime().gc();
	        }
    	}
* [AndroidExcludedRefs](https://github.com/square/leakcanary/blob/master/leakcanary-android/src/main/java/com/squareup/leakcanary/AndroidExcludedRefs.java)
	主要用于处理一些系统带来的误报泄漏，原则上是在对应版本忽略相关的泄漏。例如：
		
		ACTIVITY_CLIENT_RECORD__NEXT_IDLE(SDK_INT >= KITKAT && SDK_INT <= LOLLIPOP) {
	    @Override void add(ExcludedRefs.Builder excluded) {
	      // Android AOSP sometimes keeps a reference to a destroyed activity as a "nextIdle" client
	      // record in the android.app.ActivityThread.mActivities map.
	      // Not sure what's going on there, input welcome.
	      excluded.instanceField("android.app.ActivityThread$ActivityClientRecord", "nextIdle");
	    }
	  	}
	
* [ShortestPathFinder](https://github.com/square/leakcanary/blob/master/leakcanary-analyzer/src/main/java/com/squareup/leakcanary/ShortestPathFinder.java)
	寻找泄漏的最短强引用路径，算法类似于广度优先搜索算法，先找到对应的GCROOTS，压入队列中，依次遍历子节点并加入队列中，直到找到对应的泄漏，即可确定泄漏最短强引用路径，顺带可以返回对应泄漏是否是系统已知。
		
		Result findPath(Snapshot snapshot, Instance leakingRef) {
		    clearState();
		    canIgnoreStrings = !isString(leakingRef);
		    enqueueGcRoots(snapshot);
		    boolean excludingKnownLeaks = false;
		    LeakNode leakingNode = null;
		    while (!toVisitQueue.isEmpty() || !toVisitIfNoPathQueue.isEmpty()) {
		      LeakNode node;
		      if (!toVisitQueue.isEmpty()) {
		        node = toVisitQueue.poll();
		      } else {
		        node = toVisitIfNoPathQueue.poll();
		        excludingKnownLeaks = true;
		      }
		      if (node.instance == leakingRef) {
		        leakingNode = node;
		        break;
		      }
		      if (checkSeen(node)) {
		        continue;
		      }
		      if (node.instance instanceof RootObj) {
		        visitRootObj(node);
		      } else if (node.instance instanceof ClassObj) {
		        visitClassObj(node);
		      } else if (node.instance instanceof ClassInstance) {
		        visitClassInstance(node);
		      } else if (node.instance instanceof ArrayInstance) {
		        visitArrayInstance(node);
		      } else {
		        throw new IllegalStateException("Unexpected type for " + node.instance);
		      }
		    }
		    return new Result(leakingNode, excludingKnownLeaks);
  		}
	

## 综合比较

总结|LeakCanary
-----|------
Acvitiy泄漏|Android 4.0以上
自定义对象检测|支持
泄漏时提醒|支持
自动Dump|支持
LeakCanary优势||
Dump分析|支持，能够根据KeyedWeakReference
显示泄漏路径|显示十分详细
劣势||
白名单设置|源码写死
配合自动化|不支持
自动Fix系统泄漏|不支持，只能忽略

对于LeakCanary，泄漏问题的定位更加清晰，对于的dump文件也相对容易找到泄漏，但是如果需要非常自动化的用于自己的项目中，还需要比较大的改造。
