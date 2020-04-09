### 1. 简介

反射可以动态获取类的完整结构信息，调用对象的方法。

1. 优点：灵活性高，只有在运行时才动态创建对象实例
2. 缺点：
   1. 反射获取方法时是通过获取所有方法，然后获得的，时间开销较高
   2. 方法调用invoke()时传入的是object和object[]。不管什么类型都会转为object会生成对象，同时object[]还需要额外的封装，遍历，装箱开箱等性能消耗
   3. 每次调用方法都会检查方法等可见性，也必须检查参数等类型是否匹配
   4. 方法难以内联优化
   5. 反射是动态加载，无法JIT优化(编译执行才会用到JIT)
   6. 反射会破坏类结构，干扰类原有的内部逻辑。

### 2. 反射的使用

1. 获取Class对象：

   ```java
   //方式一：类名.class方式       
   Class<ReflectionDemo> reflectionDemoClass = ReflectionDemo.class;
   //方式二：对象.getClass()方式
   ReflectionDemo newReflectionDemo = new ReflectionDemo();
   Class<? extends ReflectionDemo> getReflectionDemo = newReflectionDemo.getClass();
   //方式三：Class.forName(包名.类名)方式
   Class<?> forNamereflectionDemo = Class.forName("xxx.xxx.ReflectionDemo");
   ```

2. 获取构造方法Constructor：

   ```java
       private static void ConstructorTest(Class c){
           try {
               //获取指定参数的构造方法，不传参则获取午餐构造。只能获取public修饰的方法
               Constructor constructor = c.getConstructor(int.class);
               //获取所有的构造，只能获取public修饰的方法
               Constructor[] constructors = c.getConstructors();
             
               //获取指定参数的构造方法，不传参则获取午餐构造。可以获取所有修饰符修饰的方法
               Constructor declaredConstructor = c.getDeclaredConstructor(int.class);
               //获取所有的构造。可以获取所有修饰符修饰的方法
               Constructor[] declaredConstructors = c.getDeclaredConstructors();
             	//获取此构造是哪个类的构造
             	Class declaringClass = constructor.getDeclaringClass();
           }catch (Exception e){
           }
   
       }
   ```

   - 获取单个构造方法时，如果有参数，需要传入参数类型.class，多个参数时按顺序填写多个，无参不传。
   - 获取构造方法名字不带Declared的，只能获取被public修饰的方法。
   - 获取方法名字不带Declared的，可以获取任何修饰符修饰的方法。
   - 通过getDeclaringClass()获取当前构造方法是哪个类的构造

3. 获取类的属性Field：

   ```java
   		private static void filedTest(Class c){
           try {
   						//获取指定的属性。传入属性名。只能获取public修饰的属性，可以获取继承的父类的属性
               Field field = c.getField("reflectionInt");
   						//获取所有的属性。只能获取public修饰的属性，可以获取父类的属性
               Field[] fields = c.getFields();
   						//获取指定的属性。传入属性名。可以获取所有修饰符修饰的属性，不可以获取继承的父类的属性
               Field reflectionInt = c.getDeclaredField("reflectionInt");
   						//获取所有的属性。可以获取所有修饰符修饰的属性，不可以获取父类的属性
               Field[] declaredFields = c.getDeclaredFields();
             	//构造一个对象
               ReflectionDemo reflectionDemo = new ReflectionDemo();
   						//通过field字段获取这个对象这个属性的值
               //如果对象被private修饰必须设置field.setAccessible(true)，负责无法访问get()会报错
           	  field.setAccessible(true);
               Object o = field.get(reflectionDemo);
             	//通过field字段设置这个对象这个属性的值
               field.set(reflectionDemo,10);
             	//获取此属性是哪个类的属性，这里返回的是ReflectionDemo
            	  Class<?> declaringClass = field.getDeclaringClass();
          }
   ```

   - 获取单个属性时，需要传入属性名
   - 获取方法名字不带Declared的，只能获取被public修饰的属性，可以获取自身及继承的属性。
   - 获取方法名字不带Declared的，可以获取任何修饰符修饰的属性，但是只能获取自身的属性，不能获取父类的属性
   - getField("字段名")获取的是字段属性，而不是具体的值。可以通过获取的field.get("对象")，获取某个对象的这个属性的值。如果对象被private修饰则必须设置field.setAccessible(true)，否则get()时会出错
   - 可以通过获取的属性field.set(对象,字段值)，来设置某个对象的某个属性的值
   - 通过getDeclaringClass()获取当前属性是哪个类的属性。(同一个类的属性调用此方法返回值是一样的)

4. 获取类的方法Method：

   ```java
       private static void methodTest(Class c) {
           try {
               //获取指定方法，第一个参数为方法名，第二个为参数类型的Class,多个参数的话，按顺序排列。只能获取public修饰的方法，可以获取继承的父类的public方法
               Method method = c.getMethod("reflectionTest", int.class);
               //获取所有方法，只能获取public修饰的方法，可以获取继承的父类的public方法
               Method[] methods = c.getMethods();
             
               //获取指定方法，第一个参数为方法名，第二个为参数类型的Class,多个参数的话，按顺序排列。能获取所有的方法，不可以获取继承的父类的public方法
               Method reflectionTest = c.getDeclaredMethod("reflectionTest", int.class);
               //获取所有方法，能获取所有的方法，不可以获取继承的父类的public方法
               Method[] declaredMethods = c.getDeclaredMethods();
             	//调用invoke()执行方法，第一个参数为执行方法的对象，第二个参数为要设置的方法的参数值
             	//如果方法被private修饰则必须设置method.setAccessible(true)，否则报错
             	method.setAccessible(true);
             	method.invoke(reflectionDemo,9);
             	//获取此方法是哪个类的方法
               Class<?> declaringClass = method.getDeclaringClass();
           } catch (Exception e) {
   
           }
   ```

   - 获取单个方法时，需要传入方法名，如果有参数，还需要参数类型.class，多个参数时按顺序填写多个。
   - 获取方法名字不带Declared的，只能获取被public修饰的方法，可以获取自身及继承的方法。
   - 获取方法名字不带Declared的，可以获取任何修饰符修饰的方法，但是只能获取自身的方法，不能获取父类的方法
   - 调用invoke()执行方法，第一个参数为执行方法的对象，第二个参数为要设置的方法的参数值，多个参数直接写多个。如果方法被private修饰则必须设置method.setAccessible(true)，否则报错。
   - 通过getDeclaringClass()获取当前方法是哪个类的方法。