title: 利用Eclipse JDT减少方法数和包大小
date: 2016-05-08 14:11:12
tags:
	- android
	- 减包
categories: tech
comments: true
---
在尝试了 [去除变量初始化赋值冗余代码](/2016/05/02/减包/分析Dalvik字节码进行减包优化/) 的方式减包后，团队减包压力依旧很大。参考了手Q团队提出的减方法数的建议后，我想到了将原有代码的private方法/变量变成package方法/变量的方案来减小包大小，主要是通过Eclipse JDT分析java语法树，不修改本地代码，只是在项目代码构建过程中动态修改java代码，从而减小包大小。最终效果是减少包大小150k左右，并且不会影响手Q本身代码的行数以及代码的可维护性，缺点是会增加项目代码构建的时间，目前是50s左右。很遗憾，最后手Q基础团队因为方案会增加构建时间的缺陷，没有采用。本篇文章主要是介绍去除private的思路以及用Eclipse JDT来实现构建过程中动态修改代码。

ps：手Q整个项目都是在RDM平台上构建的，每次构建之前都会将revert并update到最新版本，所以我才会提出在构建时动态修改java代码，可以避免修改工程本地的代码，private直接去除会给项目开发人员带来不便。
##手Q提出的减Android方法数的建议
手Q团队很早就提出了[减少Android方法数的建议](http://www.iteye.com/topic/1142147)的方案，其中针对private的建议如下：

### 避免在内部类中访问外部类的私有方法/变量
当在java内部类（包括内部匿名类）中访问外部类的私有方法/变量时，编译器会生成额外的方法，这也会增加方法数，建议编码时尽量避免。
这里是java编译器对内部类的处理方式决定的，为了让内部类能够访问外部类的数据成员，java编译器会为外部类自动生成 static Type access$iii(Outer); 方法，供内部类调用。

## 将private方法/变量变成package

参考手Q减少方法数的建议，将java原有的private方法/变量改成package，可以避免因内部类访问外部类私有方法/变量使得编译器产生额外的方法，从而减少包大小，减少方法数。该方案相对于将private变成public权限等其它方案实现起来简单，出现问题几率较小。

##常见问题与解答
###问题1：同一个包名下的类继承关系会导致private移除后出现方法重载失败。以及 同一个包名下的类继承关系会导致private移除后出现访问权限错误。
方式是通过Eclipse JDT将源代码解析成AST（抽象语法树，Abstract SyntaxTree可以用来分析代码）的语法树结构，记录了同一个包名下的方法名，如果出现同名，就不会去移除对应的private，保证了程序的鲁棒性，不会产生因重载失败和private导致构建失败的问题。

###问题2：移除private可能导致代码行数变化，不便外网追踪问题。
方式是通过Eclipse JDT分析对应的语法树，通过patch的方式去修改代码，从而只是移除private，而不会导致代码行数变化，外网如果出现问题，还是可以根据代码行数追踪问题。

###问题3：移除private是否影响反射。
否，private本身就是需要通过getDeclaredMethod 以及getDeclaredField来反射获取值的，改成package不需要调整获取方式。

###问题4：包名相同的jar包和工程代码，如果出现问题1中的private的继承问题，怎么办？。
目前没办法处理，因为AST只能分析java代码，没法分析和调整java字节码，方案是有缺陷的，会导致问题。但是一旦出问题，就会构建失败，也很容易发现问题的原因。

###问题5：将private变成package，代码不会难看么？
手Q整个项目都是在RDM平台上构建的，每次构建之前都会将代码revert并update到最新版本。基于JDT在构建时动态修改java代码，本地代码不会变化，还是有private，只是在编译时动态修改，实现减包大小和方法数。
##Eclipse JDT的使用
AST（Abstract Syntax Tree，抽象语法树）经常被用于语法分析，实现涉及到编译原理底层的东西。AST使用树状结构表示源代码的抽象语法结构，可以用于代码分析，重构等方面。很多工具尤其是编译器，都会自动将源代码转换为AST，不同的工具实现方式和定义不一样。其中Eclipse使用的就是Eclipse JDT的方式对java代码进行分析。

利用Eclipse JDT可以把java源代码变成AST抽象语法树，方便分析java类的的方法，变量等，也可以修改对应代码。入门文档可以参考[Eclipse JDT--AST入门](http://blog.csdn.net/flying881114/article/details/6187061)、[AST与ASTView简介](http://blog.csdn.net/lovelion/article/details/18953869)以及[AST的获取与访问](http://blog.csdn.net/lovelion/article/details/19050155)。
网上的相应资料比较少，大致列举如上这些就可以有些粗略的了解了。这些文档可以让你对AST的语法树，Eclipse JDT对AST的访问方式（包括ASTNode和ASTVisitor两种形式的访问方式）有些深入的了解。

##关键代码逻辑

###通过patch的方式去修改代码，以免改动代码行数

		private static final Map<String, String> formatterOptions = DefaultCodeFormatterConstants.getEclipseDefaultSettings();
	    static {
	        formatterOptions.put(JavaCore.COMPILER_COMPLIANCE, JavaCore.VERSION_1_7);
	        formatterOptions.put(JavaCore.COMPILER_CODEGEN_TARGET_PLATFORM, JavaCore.VERSION_1_7);
	        formatterOptions.put(JavaCore.COMPILER_SOURCE, JavaCore.VERSION_1_7);
	        formatterOptions.put(DefaultCodeFormatterConstants.FORMATTER_TAB_CHAR, JavaCore.SPACE);
	        formatterOptions.put(DefaultCodeFormatterConstants.FORMATTER_TAB_SIZE, "2");
			//指定代码每行代码最大字数，如果修改代码，会使单行代码变长会影响代码行数。
	        formatterOptions.put(DefaultCodeFormatterConstants.FORMATTER_LINE_SPLIT, "200");
	        formatterOptions.put(DefaultCodeFormatterConstants.FORMATTER_JOIN_LINES_IN_COMMENTS, DefaultCodeFormatterConstants.FALSE);
	        // change the option to wrap each enum constant on a new line
	        formatterOptions.put(
	            DefaultCodeFormatterConstants.FORMATTER_ALIGNMENT_FOR_ENUM_CONSTANTS,
	            DefaultCodeFormatterConstants.createAlignmentValue(
	            true,
	            DefaultCodeFormatterConstants.WRAP_ONE_PER_LINE,
	            DefaultCodeFormatterConstants.INDENT_ON_COLUMN));
	    }
		
		//按照特定的formatter格式保存代码
		public static void saveCuDiffToFile(String path,String source,CompilationUnit cu){
		    Document document = new Document(source);
			TextEdit edits = cu.rewrite(document, formatterOptions); //树上的变化生成了像diff一样的东西
			try {
				edits.apply(document);
				toFile(document.get(),path);
			} catch (MalformedTreeException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			} catch (BadLocationException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			} //应用diff
	
		}

###去除变量初始化赋值代码

		if(!ModifierSet.isFinal(iModifier) && !node.isInterface()){
			List fragments=field.fragments();
			Type type=field.getType();
			for(Object obj:fragments){
					VariableDeclarationFragment f=(VariableDeclarationFragment)obj;
					Expression ex=f.getInitializer();
					if(ex!=null && type instanceof PrimitiveType){
							if(ex instanceof NumberLiteral && ("0".equals(((NumberLiteral) ex).getToken())
									||"0.0f".equals(((NumberLiteral) ex).getToken())||"0.0F".equals(((NumberLiteral) ex).getToken())
									||"0.0d".equals(((NumberLiteral) ex).getToken())||"0.0D".equals(((NumberLiteral) ex).getToken())
									||"0.0l".equals(((NumberLiteral) ex).getToken())||"0.0L".equals(((NumberLiteral) ex).getToken()))){
								f.setInitializer(null);
							}
							if(ex instanceof BooleanLiteral && ((BooleanLiteral) ex).booleanValue()==false){
								f.setInitializer(null);
							}
					}
					if(ex!=null && type instanceof SimpleType && ex instanceof NullLiteral){
						f.setInitializer(null);
					}
			}
		}

### 去除private变量
		for(FieldDeclaration field:node.getFields()){
					int iModifier=field.getModifiers();
					if(ModifierSet.isPrivate(iModifier)){
						List modifiers=field.modifiers();
						for(Object obj:modifiers){
							if(obj instanceof Modifier){
								Modifier modifer =(Modifier)obj;
								if(modifer.isPrivate()){
									modifiers.remove(obj);
									break;
								}
							}
						}
				}
		 }

###去除private方法
对于private方法的去除需要考虑同一个包下的继承关系带来的冲突以及重写函数带来的冲突。
比如同一个包下存在如下情况，一旦都将private变成package，就会出现重写冲突，导致编译失败。
		package com.tencent;
		
		public class A extends B{
			private void test(){
				
			}
		}

		package com.tencent;
		public class B {
			 private int test(){
				return 0;
			}
		}
代码思路主要是通过
		public class ASTClass {
			public String packageName;
			public String parentName;
			public String clazzName;
			public ArrayList<String> funcs=new ArrayList<>();
		}
的结构记录每个java类，对于整个项目代码的处理分两步：

* 第一步是扫描和分析同一个包目录下java文件，得到对应的ASTClass；
* 第二步是根据第一步包目录的ASTClass扫描结果,决定当前的java文件是否需要将对应的private变成package，如果在同一个包目录下有继承关系并且存在同名函数，就不会进行修改。

但是限于JDT只能对java代码进行分析，如果包名相同的jar包和工程代码中存在上述的private方法冲突问题，确实就会导致程序被JDT优化后编译失败的情况，这里鲁棒性还是始终不够的，这是原理上的问题。
##减包代码开源以及使用
[CodeReducer的代码](https://github.com/frostpeng/CodeReducer)

###使用：
运行java程序，传入参数为优化工程的根目录，如果设置CodeReducer中的needPrintResult为true，会在运行目录下生成codelog.txt，其中记录有所有的代码修改。
##其它优化方向
目前我提出的优化主要还是在java代码级别进行修改，减少编译后的方法数和包大小，其实可以尝试用java的ASM字节码框架（基于class层面）或者ASMDex（基于dex层面）。

ASM目前的缺陷是针对class修改字节码后可能出现编译通过但是运行崩溃得去那个框，缺少有力的测试验证方式。
ASMDex 目前框架自身bug比较多，暂时也不太想去踩坑，大家有兴趣可以研究下。

##优化后感
本方案优化效果还是不错的，既可以减包大小，也能减方法数，而且能实现在编译时动态修改java代码，避免因为去掉private代码变得非常难看。但是因为动态编译以及只能在java层作优化，而无法修改jar包的代码，鲁棒性还存在一定问题，并且会导致编译时间加长，大家自行斟酌是否需要在自己的项目中使用本方案。

虽然最终这套减包方案没有在手q中应用，但在研究的过程中也学习到了很多东西。重要的是完成这件事的过程，而不仅仅是这件事的结果。