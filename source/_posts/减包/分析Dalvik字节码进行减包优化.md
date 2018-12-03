title: 分析Dalvik字节码进行减包优化
date: 2016-05-02 14:11:12
tags:
	- android
	- 减包
categories: tech
comments: true
---
Android结合版QQ空间最近几个版本在包大小配额上超标了，先后采用了包括图片压缩，功能H5，无用代码移除等手段减包，还是有着很大的减包压力。组内希望我能从代码的角度减少一些包大小，感觉有点压力山大。经过一段时间对手q安装包反编译后的Dalvik字节码的分析，发现通过调整Java代码可以减少编译后的Dalvik字节码，从而减少包大小。在这方面我做了许多的尝试，有成功有失败，拿出来给大家分享分享，多拍砖多交流。
##优化思路
* 通过dexdump反编译apk中的dex，得到对应Dalvik字节码，找到寻找冗余的字节码，尝试去除或替换冗余的字节码
* 目前主要是替换或去除原有的java代码，减少对应的Dalvik指令，从而减少安装包大小。
* 现在主要是从Dalvik字节码分析来调整Java代码，之后希望能够通过ASM等框架直接调整字节码减少现在的包大小。

##优化效果
* 去除初始化赋值方案 ————减少整个手q的发布包大小**80k**左右。
* 插桩函数优化———减少整个手q的发布包大小**2k**左右。
* 其它尝试方案，包括字符串拼接、移除interface很多空方法等，因为效果比较小、难以统一修改等问题，只是列举下分析结果，大家如果项目中出现的量比较多也是可以尝试去优化的。

##优化方案如下：
###**1、去除初始化赋值冗余**

####1.1、问题分析:
* 静态变量为类的所有对象共享，在类加载的准备阶段就会初始设置为系统零值（如下图）,比如String被设置初始值为null，而在类中存在
		
        public static String A=null;

这样的赋值行为会在之后的<cinit>()类构造器方法中执行，重复设置String A为null，增加了对应的<cinit>()方法的Dalvik指令，没有必要，可以干掉。

*	成员变量在对象创建内存分配完成后，对应的内存空间会被初始设置为系统零值(和静态变量一样)，比如int类型被设置为0,而在类中存在
		
        public int B=0;
		
这样的赋值行为会在之后的<init>()对象构造方法中执行，重复设置int B为0，增加了对应的<init>方法中的Dalvik指令，没有必要，可以干掉。
			
![系统零值](初始化赋值.jpg)

*   对于初始化赋值为系统分配默认零值的静态变量和成员变量，去掉初始化赋值，直接使用系统赋的系统零值，可以减少<cinit>和<init>中的Dalvik指令，从而减少包大小，而且可以提高类加载和对象创建的效率。	
		
        public static String A=null;	改成 public static String A；
		public int B=0;  改成 public int B;



####1.2、优化要点
*	注意对于static final的变量必须赋初值；
*	interface的变量都是static final类型的；
*   注意只有赋值为系统赋予的零值的静态变量和成员变量才能按照这种方式优化，其它比如局部变量的改动会导致编译不通过等问题。

####1.3、冗余示例：
优化前

    public class FrostTest {
			public int report_posi=0;
			public FrostTest(){
			}
	}
	
对应字节码：

    0795b4:                                        |[0795b4] com.example.frosttest.FrostTest.<init>:()V
	0795c4: 7010 5925 0100                         |0000: invoke-direct {v1}, Ljava/lang/Object;.<init>:()V // method@2559
	0795ca: 1200                                   |0003: const/4 v0, #int 0 // #0
	0795cc: 5910 c909                              |0004: iput v0, v1, Lcom/example/frosttest/FrostTest;.report_posi:I // field@09c9
	0795d6: 0e00                                   |0009: return-void

优化后：
		
    public class FrostTest {
			public int report_posi;
			public FrostTest(){
			}
	}

对应字节码

    0795b4:                                        |[0795b4] com.example.frosttest.FrostTest.<init>:()V
	0795c4: 7010 5925 0000                         |0000: invoke-direct {v0}, Ljava/lang/Object;.<init>:()V // method@2559
	0795ca: 0e00                                   |0003: return-void

减少了两行Dalvik指令的执行，最后分析结果平均优化一处可以减少安装包8个字节左右。


####1.4、优化结果：
目前在手Q6.3.0分支上利用自行写的过滤脚本（可以私下找我要对应的优化脚本用于对应的工程）可以看到优化的效果，如果对整个手q执行这个方案，预计能够**优化80k左右，修改了4677个文件，修改了17164处冗余**。




### **2、调整插桩对应的代码**
Qzone补丁包引入了插桩这一步，需要在所有qzone类的构造函数中加入对mqq.app.MobileQQ类的引用。
优化的方案是将插桩插入到对象构造函数中的语句由
		
    CtConstructor localCtConstructor = arrayOfCtConstructor[0];
    localCtConstructor.insertBeforeBody("if (com.qzone.dalvikhack.NotDoVerifyClasses.DO_VERIFY_CLASSES) System.out.print(mqq.app.MobileQQ.class);");
    localCtClass.writeFile(str);
改为
    CtConstructor localCtConstructor = arrayOfCtConstructor[0];
    localCtConstructor.insertBeforeBody("if (com.qzone.dalvikhack.NotDoVerifyClasses.DO_VERIFY_CLASSES) mqq.app.MobileQQ.class.getName();");
    localCtClass.writeFile(str);

以Qzone某个类的<init>为例，由原本的字节码

    0e640c:                                        |[0e640c] ADV_REPORT.E_REPORT_POSITION.<init>:()V
	0e641c: 7010 0b84 0200                         |0000: invoke-direct {v2}, Ljava/lang/Object;.<init>:()V // method@840b
	0e6422: 6300 1f26                              |0003: sget-boolean v0, Lcom/qzone/dalvikhack/NotDoVerifyClasses;.DO_VERIFY_CLASSES:Z // field@261f
	0e6426: 3800 0900                              |0005: if-eqz v0, 000e // +0009
	0e642a: 6200 4463                              |0007: sget-object v0, Ljava/lang/System;.out:Ljava/io/PrintStream; // field@6344
	0e642e: 1c01 2a14                              |0009: const-class v1, Lmqq/app/MobileQQ; // type@142a
	0e6432: 6e20 7883 1000                         |000b: invoke-virtual {v0, v1}, Ljava/io/PrintStream;.print:(Ljava/lang/Object;)V // method@8378
	0e6438: 0e00                                   |000e: return-void
变成了

    0e63a4:                                        |[0e63a4] ADV_REPORT.E_REPORT_POSITION.<init>:()V
	0e63b4: 7010 8183 0100                         |0000: invoke-direct {v1}, Ljava/lang/Object;.<init>:()V // method@8381
	0e63ba: 6300 0326                              |0003: sget-boolean v0, Lcom/qzone/dalvikhack/NotDoVerifyClasses;.DO_VERIFY_CLASSES:Z // field@2603
	0e63be: 3800 0700                              |0005: if-eqz v0, 000c // +0007
	0e63c2: 1c00 6714                              |0007: const-class v0, Lmqq/app/MobileQQ; // type@1467
	0e63c6: 6e10 2883 0000                         |0009: invoke-virtual {v0}, Ljava/lang/Class;.getName:()Ljava/lang/String; // method@8328
	0e63cc: 0e00                                   |000c: return-void

这里替换一处代码，将System.out.print改成getName，可以减少对象构造函数的一行Dalvik指令，替换了1314处初始化函数中插入的代码，最终将对应的qzone_plugin.apk减少了2459字节，整个手q减少2457字节左右。<font color=#FF0000>一行代码，2k收益</font>，其实还是很划算的。



### **3、字符串拼接**
下面是我针对String拼接的特殊情况“变量+""”和“""+变量”的不同形式举例分析Dalvik字节码。

	public abstract class FrostTest implements FrostInterface{
	public String a="f";
	public int b=1;
	@Override
	public void doSth1() {
		Log.i("frostpeng", a);
	}

	@Override
	public void doSth2() {
		// TODO Auto-generated method stub
		Log.i("frostpeng", a+"");
	}

	@Override
	public void doSth3() {
		// TODO Auto-generated method stub
		Log.i("frostpeng", ""+a);
	}

	@Override
	public void doSth4() {
		// TODO Auto-generated method stub
		Log.i("frostpeng", String.valueOf(a));
	}

	@Override
	public void doSth5() {
		// TODO Auto-generated method stub
		Log.i("frostpeng", String.valueOf(b));
	}
	
	public void doSth6() {
		// TODO Auto-generated method stub
		Log.i("frostpeng", b+"");
	}
	
	public void doSth7() {
		// TODO Auto-generated method stub
		Log.i("frostpeng", ""+b);
	}

	}

字节码
	
    098ee4:                                        |[098ee4] com.example.frosttest.FrostTest.doSth1:()V
	098ef4: 1a00 b715                              |0000: const-string v0, "frostpeng" // string@15b7
	098ef8: 5421 c809                              |0002: iget-object v1, v2, Lcom/example/frosttest/FrostTest;.a:Ljava/lang/String; // field@09c8
	098efc: 7120 a321 1000                         |0004: invoke-static {v0, v1}, Landroid/util/Log;.i:(Ljava/lang/String;Ljava/lang/String;)I // method@21a3
	098f02: 0e00                                   |0007: return-void
	
	098f04:                                        |[098f04] com.example.frosttest.FrostTest.doSth2:()V
	098f14: 1a00 b715                              |0000: const-string v0, "frostpeng" // string@15b7
	098f18: 2201 2f05                              |0002: new-instance v1, Ljava/lang/StringBuilder; // type@052f
	098f1c: 5432 c809                              |0004: iget-object v2, v3, Lcom/example/frosttest/FrostTest;.a:Ljava/lang/String; // field@09c8
	098f20: 7110 a225 0200                         |0006: invoke-static {v2}, Ljava/lang/String;.valueOf:(Ljava/lang/Object;)Ljava/lang/String; // method@25a2
	098f26: 0c02                                   |0009: move-result-object v2
	098f28: 7020 a525 2100                         |000a: invoke-direct {v1, v2}, Ljava/lang/StringBuilder;.<init>:(Ljava/lang/String;)V // method@25a5
	098f2e: 6e10 b125 0100                         |000d: invoke-virtual {v1}, Ljava/lang/StringBuilder;.toString:()Ljava/lang/String; // method@25b1
	098f34: 0c01                                   |0010: move-result-object v1
	098f36: 7120 a321 1000                         |0011: invoke-static {v0, v1}, Landroid/util/Log;.i:(Ljava/lang/String;Ljava/lang/String;)I // method@21a3
	098f3c: 0e00                                   |0014: return-void

	098f40:                                        |[098f40] com.example.frosttest.FrostTest.doSth3:()V
	098f50: 1a00 b715                              |0000: const-string v0, "frostpeng" // string@15b7
	098f54: 2201 2f05                              |0002: new-instance v1, Ljava/lang/StringBuilder; // type@052f
	098f58: 7010 a325 0100                         |0004: invoke-direct {v1}, Ljava/lang/StringBuilder;.<init>:()V // method@25a3
	098f5e: 5432 c809                              |0007: iget-object v2, v3, Lcom/example/frosttest/FrostTest;.a:Ljava/lang/String; // field@09c8
	098f62: 6e20 ac25 2100                         |0009: invoke-virtual {v1, v2}, Ljava/lang/StringBuilder;.append:(Ljava/lang/String;)Ljava/lang/StringBuilder; // method@25ac
	098f68: 0c01                                   |000c: move-result-object v1
	098f6a: 6e10 b125 0100                         |000d: invoke-virtual {v1}, Ljava/lang/StringBuilder;.toString:()Ljava/lang/String; // method@25b1
	098f70: 0c01                                   |0010: move-result-object v1
	098f72: 7120 a321 1000                         |0011: invoke-static {v0, v1}, Landroid/util/Log;.i:(Ljava/lang/String;Ljava/lang/String;)I // method@21a3
	098f78: 0e00                                   |0014: return-void

	098f7c:                                        |[098f7c] com.example.frosttest.FrostTest.doSth4:()V
	098f8c: 1a00 b715                              |0000: const-string v0, "frostpeng" // string@15b7
	098f90: 5421 c809                              |0002: iget-object v1, v2, Lcom/example/frosttest/FrostTest;.a:Ljava/lang/String; // field@09c8
	098f94: 7110 a225 0100                         |0004: invoke-static {v1}, Ljava/lang/String;.valueOf:(Ljava/lang/Object;)Ljava/lang/String; // method@25a2
	098f9a: 0c01                                   |0007: move-result-object v1
	098f9c: 7120 a321 1000                         |0008: invoke-static {v0, v1}, Landroid/util/Log;.i:(Ljava/lang/String;Ljava/lang/String;)I // method@21a3
	098fa2: 0e00                                   |000b: return-void

	098fa4:                                        |[098fa4] com.example.frosttest.FrostTest.doSth5:()V
	098fb4: 1a00 b715                              |0000: const-string v0, "frostpeng" // string@15b7
	098fb8: 5221 c909                              |0002: iget v1, v2, Lcom/example/frosttest/FrostTest;.b:I // field@09c9
	098fbc: 7110 a125 0100                         |0004: invoke-static {v1}, Ljava/lang/String;.valueOf:(I)Ljava/lang/String; // method@25a1
	098fc2: 0c01                                   |0007: move-result-object v1
	098fc4: 7120 a321 1000                         |0008: invoke-static {v0, v1}, Landroid/util/Log;.i:(Ljava/lang/String;Ljava/lang/String;)I // method@21a3
	098fca: 0e00                                   |000b: return-void
	
	098fcc:                                        |[098fcc] com.example.frosttest.FrostTest.doSth6:()V
	098fdc: 1a00 b715                              |0000: const-string v0, "frostpeng" // string@15b7
	098fe0: 2201 2f05                              |0002: new-instance v1, Ljava/lang/StringBuilder; // type@052f
	098fe4: 5232 c909                              |0004: iget v2, v3, Lcom/example/frosttest/FrostTest;.b:I // field@09c9
	098fe8: 7110 a125 0200                         |0006: invoke-static {v2}, Ljava/lang/String;.valueOf:(I)Ljava/lang/String; // method@25a1
	098fee: 0c02                                   |0009: move-result-object v2
	098ff0: 7020 a525 2100                         |000a: invoke-direct {v1, v2}, Ljava/lang/StringBuilder;.<init>:(Ljava/lang/String;)V // method@25a5
	098ff6: 6e10 b125 0100                         |000d: invoke-virtual {v1}, Ljava/lang/StringBuilder;.toString:()Ljava/lang/String; // method@25b1
	098ffc: 0c01                                   |0010: move-result-object v1
	098ffe: 7120 a321 1000                         |0011: invoke-static {v0, v1}, Landroid/util/Log;.i:(Ljava/lang/String;Ljava/lang/String;)I // method@21a3
	099004: 0e00                                   |0014: return-void
	
	099008:                                        |[099008] com.example.frosttest.FrostTest.doSth7:()V
	099018: 1a00 b715                              |0000: const-string v0, "frostpeng" // string@15b7
	09901c: 2201 2f05                              |0002: new-instance v1, Ljava/lang/StringBuilder; // type@052f
	099020: 7010 a325 0100                         |0004: invoke-direct {v1}, Ljava/lang/StringBuilder;.<init>:()V // method@25a3
	099026: 5232 c909                              |0007: iget v2, v3, Lcom/example/frosttest/FrostTest;.b:I // field@09c9
	09902a: 6e20 a825 2100                         |0009: invoke-virtual {v1, v2}, Ljava/lang/StringBuilder;.append:(I)Ljava/lang/StringBuilder; // method@25a8
	099030: 0c01                                   |000c: move-result-object v1
	099032: 6e10 b125 0100                         |000d: invoke-virtual {v1}, Ljava/lang/StringBuilder;.toString:()Ljava/lang/String; // method@25b1
	099038: 0c01                                   |0010: move-result-object v1
	09903a: 7120 a321 1000                         |0011: invoke-static {v0, v1}, Landroid/util/Log;.i:(Ljava/lang/String;Ljava/lang/String;)I // method@21a3
	099040: 0e00                                   |0014: return-void

从示例中可以看出各类字符串拼接方式的优劣，如果用String.valueOf()绝对是最优方案。只是通过对“变量+""”和“""+变量”的形式在手q整个项目调整以后大概能够优化6k左右,如果只是优化qzone部分，效果比较微小，脚本方面不太好过滤对应情况，暂时没有加入，只是做了下试验。
PS：其实“String +”一般来说比StringBuffer的拼接更费字节码，这个部分可以自行验证，前提是a+b+...的形式中首位a这个为变量，而不是常量，如果a是常量，则实际上和StringBuffer等同，这也是个优化点，具体可以参考文章 [从字节码视角看java字符串的拼接](http://sunqi.iteye.com/blog/2273373) 。



### **4、调整interface到class，减少实现接口造成的空方法**
很多代码中实现接口时有很多的空方法，并没有作用但还是会占用字节码，希望能够通过调整对应的interface为class，去除冗余的空方法，减少字节码，从而减少包大小。
示例如下
			
	public interface FrostInterface {
	public abstract void doSth1();
	public abstract void doSth2();
	public abstract void doSth3();
	}

	public class FrostTest1 implements FrostInterface{

		@Override
		public void doSth1() {
			// TODO Auto-generated method stub
			
		}

		@Override
		public void doSth2() {
			// TODO Auto-generated method stub
			
		}

		@Override
		public void doSth3() {
			// TODO Auto-generated method stub
			
		}
	}	

改成
	
	public abstract class FrostTest implements FrostInterface{

	@Override
	public void doSth1() {
		// TODO Auto-generated method stub
		
	}

	@Override
	public void doSth2() {
		// TODO Auto-generated method stub
		
	}

	@Override
	public void doSth3() {
		// TODO Auto-generated method stub
		
	}

	}

	public class FrostTest1 extends FrostTest{
		
	}

该方案的缺点在于修改必须手动，难度大，qzone中场景不足以引起量变，而且因为Qzone中<init>中还加入了插桩函数的负担，所以整体优化效果不佳，优化完qzone才2k不到的大小缩减，优化难度高收益小，弃坑。

后续应该还会有一些别的减包思路提出来，希望能够给一起在减包路上踩坑的朋友们一些帮助吧。
