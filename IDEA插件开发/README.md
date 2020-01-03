## 1 前言
前段时间刚从美团实习返校，手头上堆了一堆事情要忙，人直接傻掉了，学校搞工程专业认证的时候总能给你整点新花样，加之后面冬季寒冷，自己不小心又瘟了一段时间，导致一直没有产出文章，现在重感冒刚刚好，恰好自己这段时间也学了一点是有关IDEA插件开发的相关知识，在这里分享给大家。
## 2 IntelliJ插件是什么？
这么说吧，不管是IDEA也好还是咱用的Android Studio也好，其实都是基于Jetbrains IntelliJ平台的软件。而Android Studio本质上也就是IDEA+Android Gradle Plugin+IDEA Android Plugin罢了。

编写插件总的来说还是为了方便我们程序员自己的，当我们需要IDEA为我们支持更多的语言或者集成更多的开发环境的时候，就需要借助IntelliJ平台的openapi来开发插件。

当然我也不是无缘无故地去学习插件开发，我为[Gbox](https://github.com/sanyuankexie/Gbox)开发了IDEA（Android Studio）的DSL插件支持插件[handshake](https://github.com/sanyuankexie/handshake)，本文也主要围绕handshake的几个主要功能来对插件开发进行介绍，大概的功能有这些：

* 编译布局DSL
* 调试Mock布局DSL，并且做到在真机上实时预览
* DSL文件智能补全
* 自定义文件类型，识别文件名，更换图标等
* 在为IDEA添加自定义Action
* 实现类似Java自动识别main函数一样在代码旁边就有一个运行的小箭头来执行这个文件

如果你能耐心读完这篇文章，你也能轻松掌握上面所提到的这些技术。
## 3 新建IDEA插件项目
这里我建议选择的Gradle来进行开发，这样集成依赖会非常方便。记得一定要在右边勾选Interllij Platform Plugin和Java，如果想用Kotlin进行开发，下面的kotlin/JVM也要勾上（如果你跟着本文走的话，建议勾上）。

![16f5b7d01a3d436c](./assets/16f5b7d01a3d436c-1577795994585.png)

当然你也可以不选择Gradle使用上面的Interllij Platform Plugin，但是这样，你就没法用kotlin了。
![16f5b806585b4230](./assets/16f5b806585b4230-1577796020140.png)

项目建好之后，改`src/main/resources/META-INF/plugin.xml`文件，这个文件时整个插件的核心，IDEA就靠它来识别我们的插件，之后我们也还要在这个文件里注册很多与插件相关的类，但是现在我们需要填写一些插件的基本信息，反正你看哪红改哪就行了，IDEA都有提示的。

![](.\assets\16f5b8efd6733ad2.jpg)



这时候新项目里还是不包含任何代码的，我们需要在`src/main/java`和`src/main/kotlin`的文件夹下分别编写和创建`java`和`kotlin`文件。注意不要放错了，否则编译时能通过运行时会告诉你`NoClassDefineFoundError`。

## 4 PSI和VFS
在正式编写插件之前，我们首先要了解到时候我们怎么使用openapi直接访问和修改我们的源码，这里涉及到两个重要的东西Psi和VFS。

### 4.1 什么是Psi？

Psi就是程序结构接口（Program Structure Interface），用来描述IDEA代码解析器将源码解析后得到的结果的一种抽象树形结构。我们获取信息和修改文件都主要是通过Psi来进行。

但是像上面这么说还是太抽象了、太官方了，说了跟没说一样还不如叫人去看官方文档呢。所以，我以最简单的Xml的Psi为例，我画了下面这张图：
![](.\assets\16f5be265862b26c.png)
相信看完这张图你大概能应该知道Psi是个什么鬼了，Psi将源码抽象+拆分成了树型数据结构：

* PsiDictionary 是文件的目录结构，可以包含多种PsiFile，包括JavaFile，XmlFile等等
* XmlFile 是PsiFile的一种，是Xml用来描述文件，可以通过它拿到XmlDocument
* XmlDocument 可以拿到Xml的头部声明信息XmlProlog，和Xml的根XmlTag
* XmlTag 包括一组XmlAttribute和一组子XmlTag
* XmlAttribute 包括属性的名字和内容
* XmlToken 用来描述`<、=`之类的符号和文本，是最基本的组成单位，相当于编译原理中的终结符

由于网上有关插件开发的文章很少，你去啃官方文档也会发现其实官方文档屁都没说（[不相信我宁就自个去看吧，链接给宁摆这了](http://www.jetbrains.org/intellij/sdk/docs/basics/architectural_overview/psi.html)），这张图是我根据源码分析和断点调式得到结论后用[Process On](https://www.processon.com/)绘制的，全网仅此一张。

### 4.2 怎么样获取Psi？

一般来说，我们不需要主动获取Psi，而是在IDEA的某些事件触发时等待openapi将其作为参数传递给我们处理，比如：

* 下文将提到的自动补全，Psi会作为参数传递给CompletionProvider，我们可以根据传入的Psi，编写补全代码
* 下文将提到的“识别main”方法，Psi也会传给RunLineMarkerContributor和RunConfigurationProducer，让我们确认标记应该出现在哪一行和生成运行配置

当然，我们也可以主动获取Psi，但是由于Psi是跟Project相关的，主动获取Psi的API大多要求你提供Project对象作为参数，但是IDEA一般也会传给我们，你可以使用任意Psi调用`getProject()`方法，主动获取Psi的方法有以下这些：

* 通过`VirtualFileManager.getInstance().findFileByUrl()`获取VirtualFile后，从VirtualFile获取PsiFile
* 从`PsiManager.getInstance().findFile()`获取PsiFile
* 从`PsiDocumentManager.getInstance().getPsiFile()`获取PsiFile
* 或者使用`FilenameIndex.getFilesByName()`搜索项目中具有指定名称的所有文件

根据上面的那张图，我们知道PsiFile是Psi树形结构的根，拿到了PsiFile之后，我们就可将他强制类型转换为PsiFile的扩展类型比如XmlFile、JavaFile等等来对其语法格式进行访问。

当然，我们也可以通过任意一个PsiFile的子节点（比如XmlTag），在其上调用`psiElement.getContainingFile()`获取其PsiFile。

### 4.3 怎么创建Psi？

那么如果我们像主动创建Psi文件该怎么弄呢？我们可以使用`PsiFileFactory.getInstance()`，然后从使用文本创建PsiFile，创建完之后，我们还要找一个PsiDictionary调用它的add把新的PsiFile添加进去。下文我们将编写创建文件的Action，也要用到这些API。

### 4.4 怎么修改Psi？

所有的Psi都扩展自PsiElement接口，PsiElement接口最基本的修改方法有：

* `add()` 向已有的添加子节点
* `delete()` 删除当前节点
* `replace()` 替换当前节点



如果你要在其他地方修改Psi时收到通知，用`PsiManager.getInstance().addPsiTreeChangeListener()`，上面有多个回调，恰好这一块源码少见的都有注释，所以我就不展开来讲了，有兴趣的同学可以自己去研究。

### 4.5 什么是VFS？

VFS就是虚拟文件系统，与PSI不同，它是和Project无关的，所以`VirtualFileManager.getInstance()`不需要Project作为参数，它的VirtualFile更接近我们对一般文件的理解，它可以`getOutputStream()`和`getInputStream()`来对文件进行直接彻底的修改和读取，这都是Psi做不到的。

### 4.6 PSI与VFS的区别

要说PSI和VFS的区别的话，我个人理解PSI是对源码的结构化读取和修改，而VFS就是对源码的非结构化读取和修改，两者在openapi中是可以直接相互转换的，VF和通过`PsiManager.getInstance().findFile()`转换为Psi，Psi也可以直接通过`getVirtualFile()`得到VF。

## 5 自定义语言和文件格式

### Lexer



## 6 自定义Action

搞懂怎么操作Psi之后，其实你就已经可以写出一个能改自己源代码的小小插件了，只是我们还需要一个按钮去触发它，而实现这个功能的东西，就叫Action。

之前铺垫了那么多，搞了那么多看不见的东西后，咱是时候得搞点看得见得东西了。

### 6.1 什么是Action？

图中圈出的部分，都是我们开发中常用到的Action，包括新建文件，开始调试等等，都是Action。

![](./assets/16f50376820d5289.png)


### 创建Action的两种方式

第一种就是使用New→Plugin Devkit→Action来创建。

![image-20200102205308526](./assets/image-20200102205308526.png)

点击后会弹出一个窗口，让你填写该Action的相关信息。

![]()

```

```

这样创建的Action，IDEA会自动帮你在plugin.xml中进行注册，所注册的信息也就是是刚才我们所填写的信息。

![]()

另一种创建Action的办法，就是先继承AnAction，编写相关代码后，再自己到plugin.xml去注册

```

```

两种创建方法各有优劣，但其实只是步骤不同，本质上做的事情是一样的，读者可以根据自己的需要进行使用。

### 使用加载模板并使用Action创建文件



## 代码补全

### 常规做法

### 使用XmlTagNameProvider来简化工作



## 开始运行我们的DSL

### 运行配置



### 像识别java的mian函数一样识别我们的代码



### 怎样才能做到在真机上实时预览呢？

尝试用过Gbox的同学一定知道，Gbox的一个核心特性就是能够在真机上实时预览。

在同一网络环境中，通过使用Mock App扫描Studio提供的二维码，就可以在真机上实时预览布局。

当然，在最初的版本中，这个功能还是比较鸡肋的，因为一开始Gbox的Mock模块是使用AndroidStudio的控制台来绘制二维码。这在白色主题下的Studio工作是完全正常的，但如果在黑色主题下，由于黑白颠倒，所以zxing库就无法识别了，导致有些同学扫码之后出错后者没反应，到最后不得不在后面的版本的控制台输出中加上让使用者更换Studio颜色的提示，这种体验实际上是相当糟糕的。

![](./assets/16e4128ca0f97d4b.png)

所以现在的版本将控制台绘制二维码，改为用可控的swing窗口进行绘制，无论Studio用什么主题二维码都可以被正常扫描。

![新二维码图片]()

这个窗口的原理很简单，其实就是管理了一个JFrame的生命周期罢了，二维码是用Google的zxing生成的，然后搞了一个awt的BuffedImage渲染上去，纯属调API操作，所以我就直接贴代码了：

[src/main/kotlin/com/guet/flexbox/handshake/mock/QrCodeForm.kt](https://github.com/sanyuankexie/handshake/blob/master/src/main/kotlin/com/guet/flexbox/handshake/mock/QrCodeForm.kt)

```kotlin
class QrCodeForm(url: String) : JFrame() {

    init {
        iconImage = IconUtil.toImage(fileIcon)
        val size = 300
        title = "For playground"
        defaultCloseOperation = DISPOSE_ON_CLOSE
        setSize(size, size)
        isResizable = false
        val content = contentPane
        val panel = ImageView()
        AppExecutorUtil.getAppExecutorService().execute {
            val image = generateQR(url, size)//异步生成图片
            EventQueue.invokeLater {
                panel.image = image//发回主线程
            }
        }
        panel.setSize(size, size)
        content.add(panel)
        val kit = Toolkit.getDefaultToolkit() //定义工具包
        val screenSize = kit.screenSize //获取屏幕的尺寸
        val screenWidth = screenSize.width //获取屏幕的宽
        val screenHeight = screenSize.height //获取屏幕的高
        setLocation(screenWidth / 2 - size / 2, screenHeight / 2 - size / 2)//设置窗口居中显示
        isVisible = true
        isAlwaysOnTop = true
        val cancel = object : WeakReference<JFrame>(this), Runnable {
            override fun run() {
                get()?.isAlwaysOnTop = false
            }
        }
        AppExecutorUtil.getAppScheduledExecutorService().schedule({
            EventQueue.invokeLater(cancel)
        }, 100, TimeUnit.MILLISECONDS)
    }


    companion object {
        /**
         * Generating a qr code with provided content
         *
         * @param content The content that should be in the QR
         * @return An Buffered image object containing the qr code
         */
        private fun generateQR(content: String, size: Int): BufferedImage {
            try {

                val hintMap = HashMap<EncodeHintType, ErrorCorrectionLevel>()
                hintMap[EncodeHintType.ERROR_CORRECTION] = ErrorCorrectionLevel.L
                val qrCodeWriter = QRCodeWriter()
                val byteMatrix = qrCodeWriter.encode(
                    content,
                    BarcodeFormat.QR_CODE, size, size, hintMap
                )
                val width = byteMatrix.width
                val image = UIUtil.createImage(
                    width,
                    width,
                    BufferedImage.TYPE_INT_RGB
                )
                val graphics = image.createGraphics() as Graphics2D
                graphics.color = JBColor.WHITE
                graphics.fillRect(0, 0, width, width)//先全部涂白
                graphics.color = JBColor.BLACK

                for (i in 0 until width) {
                    for (j in 0 until width) {
                        if (byteMatrix.get(i, j)) {
                            graphics.fillRect(i, j, 1, 1)//为true的涂一个小黑方块
                        }
                    }
                }
                return image
            } catch (e: WriterException) {
                throw RuntimeException(e)
            }
        }
    }
}
```

嗯？怎么说了这么多还是没说到怎么实时在真机上预览布局呢？别着急，秘密就藏在这个二维码里。这个二维码实际上是一个http协议的url，当你点击运行按钮加载运行生成运行配置的时候，实际上使用电脑开启了一个http服务器，这个url就是`http://<你的局域网IP>:<端口号默认8080>`。当使用Mock App扫描这个二维码之后，App获取到url，就可以通过请求每秒向电脑发送http请求，将最新的布局文件和数据拉下来，然后在真机上重新渲染，这样就实现了实时预览。

![]()

现在接着上面没讲完的RunProfileState现在我们要来实现它，用ProcessHandler来管理上面的QrCodeForm。

```java

```

没错！这里我违背了jetbrains的推荐做法，没有把mock服务器和窗口放在一个独立的外部进程中去运行！至于为什么要这么做，我可以光明正大地告诉你纯粹是为了偷懒、为了方便、为了能用就行！但其实也不是，因为开启mock服务器这个程序实在是太轻量级了，而且又和插件密不可分，所以把它们打包，并且在一起运行在同一进程，才是最好地选择，并不全是因为我是一条懒狗。

**PS**：但是不建议大家这么做啊，特别是当你要运行的调试程序本来就是一个外置程序的时候，不建议大家把代码内置，而应该使用类似继承`BaseProcessHandler`之类ProcessHandler来启动外部程序。

## openapi中一些常用的类汇总

### 共用线程池

### PsiTreeUtil

### openapi中的Manager

* VirtualFileManager



## 其他注意事项

插件使用独立的ClassLoader进行加载，【声明依赖】

注意你的目录kotlin和java的文件不能混淆，除非你在build.gradle中手动指定sourceSet




## 更新插件对GBox有什么影响？
显而易见的拥有插件之后开发Gbox的布局变得更快了，并且调试和编译都很方便，还很容易的就能将布局代码和VCS集成。当然Gbox的插件还有很多不足，例如由于我对openapi的研究有限，现在还没法实现attributeValue中的智能补全，待今后继续研究之后，会慢慢加上。

## 参考资料
国内涉及插件开发的文章非常稀有，而且由于openapi的源码是几乎没有任何注释的，导致很多时候明知道某个功能能实现但却不知道怎么写就非常蛋疼。自己在开发的时候参考了阿里前辈moxun（有`xxx@apache.org`邮箱的大佬）开发的weex的IDEA插件以及读了不知道多少IDEA的源码之后才不怎么顺利的开发出来（但其实也就花了三四天的样子，openapi代码质量很高，但很多源码没注释没文档就是感觉恶心）。

下面本编文章和插件开发中的是主要的参考资料。

* weex插件作者写的文章《[怎么找到一个好名字idea插件开发](https://www.cnblogs.com/liqiking/p/6792991.html)》
* Jetbrains官方文档《[IntelliJ Platform SDK](http://www.jetbrains.org/intellij/sdk/docs/welcome.html)》

当然IntelliJ Platform的openapi很庞大，本文所涉及的只是开发Gbox插件要用到的很小的一部分，很多有关ui、lexer、bnf等相关技巧就需要读者进一步去读文档和源码了。

## 获取相关源码

* 追求性能的动态布局框架【[Gbox](https://github.com/sanyuankexie/Gbox)】
* 与**Gbox**配套的IDEA插件【[handshake](https://github.com/sanyuankexie/handshake)】
* weex的IDEA插件【[weex-language-support](https://github.com/misakuo/weex-language-support)】

## 结语&关于我

之前在掘金上发沸点，可能是因为我说话太骚了，居然有人质疑我是骗子，偏赞的那种，我也是醉了，虽然后来澄清了，不过还是在这里解释一下，我是应届生，虽然拿了offer，但是人还没毕业入职呢。自己很关心前沿技术和核心技术的发展，所以也会写一些所谓的技术文章，热衷于分享和开源，对自己学过的技术不会保留，只是可能没事间码那么多字罢了（说到底还不是因为我是一条懒狗）。