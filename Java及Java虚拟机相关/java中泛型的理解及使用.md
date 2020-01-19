### 1. 泛型的一些概念

Java从1.5开始加入了泛型，主要是解决类型安全及扩展问题，它的本质是参数化类型的应用，也就是说所操作的数据类型被指定为一个参数。这种参数类型可以用在类，接口和方法的创建中，分别称为泛型类，泛型接口和泛型方法。

泛型的一些特性：

* 泛型是类型擦除的：Java的泛型只在编译期有效，它只在程序源码中存在，在编译后的字节码文件中就已经替换为原来的原生类型。并且在对象的地方插入了强制转型代码。因此对于运行期的Java语言来说List<String>和List<Int>就是同一个类。

* 泛型识别：为了解决泛型的识别问题，Java引入了Signature,LocalVaribaleTypeTable等新的属性用于解决参数类型的识别问题。Signature的作用就是存储一个方法在字节码层面的特征签名，这个属性中保存的参数类型并不是原生类型，而是包括了参数化类型的信息。从Signature属性可以得知，所谓的擦除仅仅是对方法的Code属性中的字节码进行擦锁，实际上元数据中还是保留了泛型信息，所以还是通过class对象反射得到泛型信息。

* 泛型不是协变的：如String是Object的子类，但是List<String>并不是List<Object>的子类型，它们是2个单独的类型。

### 2. 泛型的使用

**泛型类及泛型接口**

```
public class Shop<T> {
  	//商品数量
  	int count;
  	//抽象商品
    T goods;
    public T get(){
        return t;
    }
}

```
假如有一个Shop类，现在并不明确是什么类型的店，里面定义了卖物品的数量int类型的count,以及不确定什么类型的商品goods。

这里有个问题，不确定类型，但是必须有类型。否则编译器就会报错。所以要给goods一个类型，因为不确定，所以可以先用一个代号代替，等实际类型确定时再替换掉，这里使用T。但是这个T只是一个代号，编译器不可能认识，所以要在类名的右边用<>包裹T，告诉编译器T是一个类型的代号，然后就可以在类中使用T代表某种类型了。

这里的T不具备特殊意义，只是到时会被替换的实际类型的代号，所以它可以随便写成A，B，C，D等等其他任何形式。

再继续看，泛型T可以代表任何类型，这个范围太广，如果要限定是某一种类型，或符合某些特性呢？，如是一家书店，所以如果要限定某种类型的话就要对T进行限制：

```
public class Shop<T extends BookShop> {
    
    //商品数量
    int count;

    //抽象商品
    T goods;

    public T get(){
        return goods;
    }
}
```

使用extends关键字，限定了T只能是BookShop类型，那么如果要求即是书店，又是咖啡店呢：

```
//如果有2种特性，直接用&符号连接,BookShop为类，CoffeShop为接口
public class Shop<T extends BookShop & CoffeShop> {

    //商品数量
    int count;

    //抽象商品
    T goods;

    public T get(){
        return goods;
    }
}
```

直接使用&符号连接即可，这里的extends遵循Java的单继承原则，BookShop和CoffeShop只能一个是类，另外个是接口，当然接口是可以多实现的，继续用&连接即可，如添加Rest属性：

```
//如果有2种特性，直接用&符号连接,BookShop为类，CoffeShop，Rest为接口，遵循Java的单继承，多实现原则
public class Shop<T extends BookShop & CoffeShop & Rest> {

    //商品数量
    int count;

    //抽象商品
    T goods;

    public T get(){
        return goods;
    }
}
```

再来看，假如要开一个分店，BranchShop:

```
public class BranchShop<B> extends Shop{
    
    int count ;
    
    B goods;

    @Override
    public BookShop get() {
        return super.get();
    }
}
```

可以看到BranchShop声明了自己的泛型B（写在类右边，用<>包裹）但是重写的get()返回值类型变成了BookShop，如果BranchShop需要get()的返回值类型是泛型类型呢？：

```
public class BranchShop<B> extends Shop{

    int count ;

    B goods;
		//这里不会报错
    public B getGoods(){
        return goods;
    }
    @Override
  	//这里会报错
    public B get() {
      	//这里也会报错
        return super.get();
    }
}
```

getGoods()不会报错，但是get()会报错，因为get是父Shop的方法，而Shop的泛型T是这样的:

```
T extends BookShop & CoffeShop & Rest
```

它限制了范围，所以如果要重写父类的方法，也要和父类的泛型保持一样的限制（也可以不重写，不限制，如getGoods()）:

```
public class BranchShop<B extends BookShop & CoffeShop & Rest> extends Shop{

    int count ;

    B goods;

    public B getGoods(){
        return goods;
    }
		
    @Override
  	//这里不报错了
    public B get() {
      //这里还是报错
        return super.get();
    }
}
```

可以看到当B和父类的T同步了限制之后，返回值类型不报错了，但是super.get()还是报错，这是因为之前说过，要使用泛型必须在类的右边声明，BranchShop在自己右边声明了<B extends BookShop & CoffeShop & Rest>，但是在BranchShop中，父类Shop并没有声明，没有声明就没法使用，这里子类把父类的泛型类型丢失了，所以要给它加上声明:

```
public class BranchShop<B extends BookShop & CoffeShop & Rest> extends Shop<B>{

    int count ;

    B goods;

    public B getGoods(){
        return goods;
    }

    @Override
    public B get() {
        return super.get();
    }
}

```

直接在Shop后声明<B>就OK，这里的泛型就不能随便写，必须和BranchShop的声明保持一致，也就是必须写成B。

> A：public class BranchShop<B extends BookShop & CoffeShop & Rest> extends Shop<B>
B：public class BranchShop<B> extends Shop<B extends BookShop & CoffeShop & Rest>
至于Shop<>,里只能和BranchShop保持一致，以及为什么不写成B，而写成A这样，可以简单理解为，泛型必须声明在类的右边且必须声明才能使用。所以Shop<>它是使用泛型，而不是声明，所以不能随便写成一个没有声明的类型，所以只能是B。所以限制条件也必须加在BranchShop后，也就是A写法。


**2. 泛型方法**

如果类没有声明泛型类型，那么如何在方法里使用泛型？，看一下一个常用的方法findViewById()：

```
    public <T extends View> T findViewById(@IdRes int id) {
        return this.getDelegate().findViewById(id);
    }
```
可以看到，是在返回值前面声明了T，并且限制了T extends View,然后返回值就可以使用T了。


**3. 通配符**

* 上界通配符

```
public class FruitShop {
		//Apple集合，Apple继承自Fruit
    List<Apple> apples;
	  //Orange集合，Orange继承自Fruit
    List<Orange> oranges;
		//这样写没法遍历Apple和Orange集合
    public  void getFruit(List<Fruit> fruits){
    }
}
```

假如有一个水果店，需要一个通用的方法遍历每一种水果，但是由于泛型不是协变的，所以上面的getFruit()是没法遍历Apple和Orange集合的，这里要解决这个问题，要用到? extends关键字：

```
public class FruitShop {

    List<Apple> apples;

    List<Orange> oranges;

    public  void getFruit(List<? extends Fruit> fruits){
        for (int i = 0; i < fruits.size(); i++) {
          	//得到每一个fruit
            Fruit fruit = fruits.get(i);
        }

    }
}
```

使用? extends Fruit代表了Fruit某个字类型，所以只要是Fruit子类型都可以（每个类型都是自身的子类型）进行遍历。

但是上界通配符有一个问题：

```
public class FruitShop<E> {

    List<Apple> apples;

    List<Orange> oranges;

    public  void getFruit(List<? extends Fruit> fruits){
      	//不报错
        fruits.get(0);
    		//这里报错
        fruits.add(new Orange());
    }
}
```

在getFruit()接收了一个Fruit一个子类型的List，但是只知道是Fruit的子类型，具体是哪种子类型是不知道的，如上图，很可能传入了一个Apple的的List，但是有可能添加进一个Orange，所以Java不允许这样操作。

**这就是上界通配符？extends T,限制了？只能是T或者T的子类，并且只能取内容，不能存内容**

* 下界通配符

```
    public void setFruit(List<? super Fruit> fruits){
        fruits.add(new Apple());
        Object object = fruits.get(0);
    }
```

和上界通配符刚好相反，下届通配符代表可以接受Fruit及它的所有超类型。
它的特性是可以存东西，但是不能取，或者只能用Object接收，但是类型信息会全部丢失。因为确定取的到底是什么类型。

* 无界通配符

```
   	//无界通配符
    public void setFruit(List<?> fruits){
      	//这行报错
        fruits.add(new Apple());
      	//同样只能用Object接收
        Object o = fruits.get(0);
    } 
```
？代表不进行任何限制，可以是任何类型。所以它既不能存，也不能取（或者取出的值元素只能用Object接收）。

但是可以通过写一个帮助类，来达到让？能取的功能：
```
    public void setFruit(List<?> fruits){
      	//这行报错
        fruits.add(fruits.get(0));
      	//只能获取到Object类型
        Object o = fruits.get(0);
        setFruitHelper(fruits);
    }

    public <E> void setFruitHelper(List<E> e){
      	//这行不报错
        e.add(e.get(0));
      	//可以获取到T类型
        T t = e.get(0);


    }
```
setFruitHelper()知道e列表中取出的任何值均为E类型，并且知道E类型的任何值放进列表都是安全的。

无界通配符也有要注意的问题：

```
    public  void test(){
    //构造一个下界通配符集合
    List<? super Fruit> f =new ArrayList<>();
     //添加Apple
     f.add(new Apple());
     //添加Orange
     f.add(new Orange());
      //把下届通配符集合传入无界通配符集合里
      setFruit(f);
    }
	
    public void setFruit(List<?> fruits){
      	//调用Helper方法
        setFruitHelper(fruits);
    }

    public <T> void setFruitHelper(List<T> e){
      	//在这里执行get()以及add()
      
      	//取出的是Apple
        T t = e.get(0);
      	//添加的是Apple
        e.add(e.get(0));
      	//取出的是Orange
        T t = e.get(1);
      	//添加的是Orange
        e.add(e.get(1));

    }
```
可以看到通过setFruitHelper()，规避了下界通配符不能取的问题。使用的时候需要注意。

### 3. 获取泛型类型

**前面说过，泛型在运行期间是擦除的，但是会保存在class文件里，所以可以从class里获取**

**1. 获取类及接口泛型，Java提供了2个关于泛型的方法：**

* getGenericSuperclass：返回此类所表示的实体的直接超类的类型，看一下代码

```
//part1:AppleShop自身的泛型T，明确了父类FruitShop的泛型Apple
public class AppleShop<T> extends FruitShop<Apple> {

}

//part 2
//获取AppleShop的class
Class<AppleShop> appleShopClass = AppleShop.class;
//获取type
Type genericSuperclass = appleShopClass.getGenericSuperclass();
//获取具体type的数组结合
Type[] actualTypeArguments = ((ParameterizedType)genericSuperclass).getActualTypeArguments();


```
可以看到在part1中，AppleShop类声明了自身的泛型T，明确了父类的泛型类型为Apple

然后在part2中，先获取了AppleShop的class，class通过调用getGenericSuperclass()，返回了Type。

然后通过type再获取具体的泛型数组，由于这里只有一个泛型，所以取第一个actualTypeArguments[0],分别看一下它们的结果：

```
type:xxx.xxx.xxx.FruitShop<xxx.xxx.xxx.Apple>
actualTypeArguments[0]:xxx.xxx.xxx.Apple
```
可以看到type包含了父类FruitShop信息，actualTypeArguments[0]返回了具体泛型类型。

这里要注意一点，这个方法获取的是明确的父类泛型，不是自身声明的泛型类型。因为自身泛型类型这时还并没有确定。

* getGenericInterfaces：返回此类直接实现的所有接口类型，看一下代码：

```
//part1:明确了父类FruitShop的泛型Apple
public class AppleShop implements Rest<Apple> {

}
//part2:
//获取AppleShop的class
Class<AppleShop> appleShopClass = AppleShop.class;
//获取type数组（因为可能实现多个接口，所以是数组）
Type[] genericInterfaces = appleShopClass.getGenericInterfaces();
//取第一个的type
ParameterizedType genericInterface = (ParameterizedType) genericInterfaces[0];
//获取具体type的数组结合
Type[] actualTypeArguments = genericInterface.getActualTypeArguments();
```
看一下获取结果：

```
type:xxx.xxx.xxx.FruitShop<xxx.xxx.xxx.Apple>
actualTypeArguments[0]:xxx.xxx.xxx.Apple
```
和getGenericSuperclass的过程和结果都是一样的。

**2. 获取方法和成员变量的泛型**

方法及成员变量的泛型，可以通过反射获取。

```
public class AppleShop<T> extends FruitShop<Apple> {
		//成员变量泛型
    List<Apple> apples = new ArrayList<>();
		//方法泛型，包括返回值泛型以及参数泛型
    public List<Apple> getApples(List<Apple> count){
       
        return apples;
    }

}
```

看一下如何获取的代码：

```
//获取class
        Class<AppleShop> appleShopClass = AppleShop.class;
        try {
            //获取apples成员变量，这里只是测试，使用了获取单个成员变量的方法
            Field apples = appleShopClass.getDeclaredField("apples");
            //获取成员变量的type
            Type genericType = apples.getGenericType();
            //获取成员变量泛型的具体type
            Type[] actualTypeArguments = ((ParameterizedType) genericType).getActualTypeArguments();

        }catch (Exception e){

        }

        try{
            //获取getApples(),这里只是测试，使用了获取单个方法的方法
            Method getApples = appleShopClass.getMethod("getApples", new Class[]{List.class});
            //获取方法参数的Type集合
            Type[] genericParameterTypes = getApples.getGenericParameterTypes();
            //取第一个参数的具体Type
            ParameterizedType genericParameterType = (ParameterizedType) genericParameterTypes[0];
            //取第一个参数的具体类型集合
            Type[] actualTypeArguments1 = genericParameterType.getActualTypeArguments();
            
            //获取getApples()的返回值Type
            Type genericReturnType = getApples.getGenericReturnType();
            //获取返回值具体类型的Type集合
            Type[] actualTypeArguments = ((ParameterizedType) genericReturnType).getActualTypeArguments();

        }catch (Exception e){

        }


```
以上就是获取成员变量以及方法返回值，方法参数泛型类型的方式。

**3. 运行时代码的泛型**

前面的都是通过class获取的泛型类型，但是有些代码是运行时才确定的，如下：

```
    public List<Apple> getApples(List<Apple> count){
    		//这里运行时才会确定
        List <Orange> oranges=new ArrayList<>();
        return apples;
    }
```
运行时才能确定的，通过之前的方法就没法获取。

还是前面说过,class会保留泛型信息，那么这里通过建立匿名内部类的方式，然后就会产生class文件，从而让泛型类型保存起来，如下：

```
  public List<Apple> getApples(List<Apple> count){
  			//通过在后面加一个{}的方式建立一个匿名内部类
        List <Apple> applesList=new ArrayList<Apple>(){};
    		//获取type,和之前一样
        Type genericSuperclass = applesList.getClass().getGenericSuperclass();
        return apples;
    }
```
通过建立一个匿名内部类的方式，产生class文件，然后就可以获取泛型类型了。



