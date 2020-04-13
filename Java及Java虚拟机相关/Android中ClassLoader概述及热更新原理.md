**Android中的重点ClassLoader包括以下几类：**

1. BootClassLoader:系统启动时会使用BootClassLoader来预加载常用类，在程序中无法直接调用

2. DexClassLoader:可以加载dex文件，以及包含dex的压缩文件(apk和jar文件)，不管加载哪种文件，最终都要加载dex文件。可以从SD卡中安装未安装的apk

   ```java
   public class DexClassLoader extends BaseDexClassLoader {
   			//dexPath:dex相关的路径集合，多个路径用文件分割符“：”分隔
     		//optimizedDirectory解压dex文件存储路径，这个路径必须时一个内部存储路径，一般情况下，使用当前应用程序的私有路径：/data/data/<Package Name>/...
     		//librarySearchPath:包含c/C++库的路径结合多个路径用文件分割符“：”分隔，可以为null
     		//parent:父加载器
          public DexClassLoader(String dexPath, String optimizedDirectory,
                  String librarySearchPath, ClassLoader parent) {
              super(dexPath, null, librarySearchPath, parent);
          }
      }
   ```

   DexClassLoader继承自BaseDexClassLoader，方法都在BaseDexClassLoader中实现

3. PathClassLoader:Android系统使用PathClassLoader来加载系统类和应用程序的类。

   ```java
      public class PathClassLoader extends BaseDexClassLoader {
          public PathClassLoader(String dexPath, ClassLoader parent) {
              super(dexPath, null, null, parent);
          }
   			//dexPath:dex相关的路径集合，多个路径用文件分割符“：”分隔
     		//librarySearchPath:包含c/C++库的路径结合多个路径用文件分割符“：”分隔，可以为null
     		//parent:父加载器
          public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
              super(dexPath, null, librarySearchPath, parent);
          }
      }
   ```

   PathClassLoader继承自BaseDexClassLoader，方法都在BaseDexClassLoader中实现。在PathClassLoader中没有optimizedDirectory，这是因为已经默认设置了optimizedDirectory的值为data/dalvik-cache,所以PathClassLoader无法定义解压dex文件存储路径，因此PathClassLoader通常用来加载已经安装的apk的dex文件(安装的apk的文件会存储在/data/dalvik-cache中)

4. InMemoryDexClassLoader:Android8.0新增的类加载器，用于加载内存中的dex

5. BaseDexClassLoader:其他classLoader都继承自BaseDexClassLoader，并且它们的构造都调用了父类的构造，那么看一下它的构造：

   ```java
        public BaseDexClassLoader(String dexPath, File optimizedDirectory,
                 String librarySearchPath, ClassLoader parent, boolean isTrusted) {
             super(parent);
          		//初始化DexPathList
             this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);
   
         }
   ```

   在这里初始化了DexPathList，并传入了dexPath，看一下代码：

   ```java
        DexPathList(ClassLoader definingContext, String dexPath,
              // private Element[] dexElements
              this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                                 suppressedExceptions, definingContext, isTrusted);
   
          }
   ```

   在这里又初始化了一个Element[]，这一步其实就是将所有的dex文件存储到Element[]中。

   **所以，Android中ClassLoader构造完成后同时构造了一个Element数组，它里面存储了所有的DexFile。**

**类加载过程：**

类加载遵循双亲委托模式，加载时首先判断该class是否已加载，如果没有，则委托父类加载器进行查找，这样依次递归，直到委托到顶层的BootClassLoader，如果BootClassLoader找到了该class，就直接返回，如果找 不到，则依次向下查找，如果还没找到，最后会交由自身查找。

类加载器通过loadClass()方法来加载类，这个方法在ClassLoader中实现，看一下它的loadClass():

```
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            //先去获取类是否已加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
            		//如果还没加载
                try {
                    if (parent != null) {
                    //如果父加载器不为null，则代表当前不是最顶层的ClassLoader，则向上传递，让父类去加载
                        c = parent.loadClass(name, false);
                    } else {
                    //如果父加载器为null,则代表已到顶层ClassLoader，直接去加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
								//如果父类没加载到，那么自身去加载
                if (c == null) {
                    c = findClass(name);
                }
            }
            return c;
    }
```

可以看到，如果是自定义classLoader的话，那么最后调用的是findClass()去加载类，DexClassLoader和PathClassLoader都继承自BaseDexClassLoader，findClass()也是在BaseDexClassLoader中实现，看一下它的实现：

```
     @Override
       protected Class<?> findClass(String name) throws ClassNotFoundException {
				
           Class c = pathList.findClass(name, suppressedExceptions);

           return c;
       }
```

之前总结过，pathList是一个DexPathList，那么看一下它的findCLass():

```
       @Override
       public Class<?> findClass(String name, List<Throwable> suppressed) {
       		//遍历Element，每一个element里有一个DexFile
           for (Element element : dexElements) {
           //从dex中查找class
               Class<?> clazz = element.findClass(name, definingContext, suppressed);
               if (clazz != null) {
               //如果找到，则直接返回
                   return clazz;
               }
           }
           return null;
       }
		
    	//element的findClass()     
			public Class<?> findClass(String name, ClassLoader definingContext,
                   List<Throwable> suppressed) {
        	//这里最终会调用DexFile中的native方法去加载一个class
               return dexFile != null ? dexFile.loadClassBinaryName(name, definingContext, suppressed)
                       : null;
           }
      
```

可以看到，最终是通过遍历Element数组，每一个element内部有一个DexFile，通过这个DexFile去加载返回class。如果返回的class不为null，则直接返回。

**总结：**

- ClassLoader构造后就自动构造了DexPathList，同时在DexPathList的构造中又构造了一个Element数组，里面存储了所有DexFile。
- 加载一个类时会遍历Element数组，得到每一个DexFile,通过DexFile会加载类，如果找到则直接返回，也就是说找到后不再去遍历后面的dex

- 自定义ClassLoader时一般使用PathClassLoader，PathClassLoader中默认了dex文件解压目录，无法自定义。PathClassLoader在Zygote启动SystemServer时创建。
- 自定义ClassLoader需要重写findClass()去加载自己需要加载的类。

**Android中热修复原理：**

1. 类加载方案：加载Class最终是通过遍历Element得到每一个DexFile,然后DexFile再去加载类，找到后直接返回，不再继续寻找。那么假如要修改一个出错的类A，在修改错误后，把新的类A打包成dex,然后通过反射的方式把这个dex放到Element数组的第一个位置，那么再次加载时，遍历到第一个dex时就会找到修改后的A类，不会再去往后找错误的类A，达成修复的目的。这个方案的缺点是必须重启App才有效，因为类一旦被加载是无法被卸载的，如果不重启，那么会通过findLoadedClass得到已加载的错误的class。
2. 底层替换方案：在native层修改，比如替换掉整个方法体
3. So的修复
   - 将so补丁插入到NativeLibraryElement数组的前部，让so补丁的路径先被返回和加载
   - 调用System的load()来接管so的加载入口
4. 资源修复
   - 创建新的AssetManager,通过反射调用addAssetPath方法加载外部资源，这样新创建的AssetManager就含有了外部资源
   - 将AssetManager类型的mAssets字段引用全部替换为新建的AssetManager