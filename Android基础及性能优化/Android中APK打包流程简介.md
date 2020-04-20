**1. APK简介：**

APK其实就是一个压缩包，它里面包括：

- classes.dex:Dex是Android平台上的可执行文件，Android虚拟机Dalvik/Art支持的字节码文件格式。通过Android平台上的工具可以将Java字节码转换为Dex字节码，转换时会压缩常量池，消除冗余信息等。同时.class是基于栈而.dex是基于寄存器的，更适合移动端。
- res:资源文件夹
- resources.arsc:这个文件记录了所有应用程序资源目录的信息，包括每一个资源名称，类型，值，ID以及所配置的维度信息。resources.arsc是资源索引表，可以在给定资源ID和设备配置信息的情况下，能够在应用程序目录中快速找到最匹配的资源。
- lib:存放so动态链接库。
- META-INF:签名文件夹，存放了3个文件。2个是对资源文件进行SHA1处理，一个是签名和公钥证书
- AndroidManifest.xml:是每个应用程序必须的文件，描述了package中的组件，如Activity，service等，及它们的启动模式等。还能制定permissions(权限控制)和instrumentation(测试)

**2. 打包流程：**

- 使用aapt打包资源文件，生成R.java文件：

  - 首先检查AndroidManifest.xml的合法性，然后对res目录下的资源目录进行处理。处理内容包括文件名的合法性检查，向资源表添加条目等。

  - 编译res与asserts目录下的资源生成resource.arsc文件。然后生成R.java文件。

    > resource.arsc：清单文件，是一个资源索引表，可以方便地让应用程序运行时根据ID来找到不同的设备分辨率对应的资源。主要面向程序。
    >
    > R.java：appt工具对每个资源文件都生成了唯一ID(除了assets外)，这些ID保存在R.java中。主要面向程序员。

  - 编译res目录下的xml，编译过的xml文件就简单的被加密。最后将所有资源与编译生成的resource.arsc文件，以及加密AndroidManifest.xml打包压缩成resources.ap_文件。除了assets和res/raw资源被原封不懂地打包进APK之外，其他资源都会被编译或者处理。

    > 将xml编译成二进制文件，占用空间会更小。因为所有的xml元素的标签，属性名称，属性值和内容所涉及到的字符串都会被统一收集到一个字符串资源池中并去重。有了这个字符串资源池，使用字符串的地方就会被替换成一个整数索引值，从而减小文件的大小。同时由于二进制的XML元素里不再包含字符串值，避免了字符串解析，速度也会提高。

  aapt传统的打包主要是指res和Java代码的打包，aapt打包走的是单线程。传统的aapt会执行2次，第一次是生产R.java，参与javac编译，第二次是对res里面的文件进行编译。最后将Dex文件与编译好的资源文件打包成apk,进行签名。整个流程没有任务缓存，没有并发，没有增量，每次构建都是一个全新的流程。因此每次构建时间都比较恒定，代码量，资源量越多，构建的时间就越慢。

- 使用AIDL工具处理AIDL文件，生成对应的java文件

- 编译工程源码，生成相应的class文件：使用javac工具编译src目录下的所有源文件，生成class文件。这些源文件包括R.java,AIDL生成的java文件，库jar文件。

- 生成dex文件：使用dx工具将class文件转换为dex文件。转换时会压缩常量池，消除冗余信息等。

- 打包生成apk：使用sdklib的ApkBuilderMain将打包后的资源文件,dex文件，lib文件打包成APK

- 对apk进行签名：Android的应用程序需要签名才能安装。调试时会默认使用debug.keystore对apk进行签名

- 对签名后的apk进行签名处理：使用zipalign工具对apk进行对齐处理

  > 对齐处理主要是将文件的起始位置偏移为4字节的整数倍，这样通过内存映射访问apk时的速度会更快。
  >
  > 因为如果每个资源的开始位置都是上一个资源之后的4n字节。那么访问下一个资源就不用遍历，直接跳到4n字节处即可。