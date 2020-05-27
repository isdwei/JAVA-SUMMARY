# JAVA基础知识总结

## Object类

### hashCode方法与equals方法

[转载](https://www.cnblogs.com/wang-meng/p/7501378.html)

1.  原生的hashCode值是根据内存地址换算出来的一个值。
2.  原生的equals方法是严格判断一个对象是否相等的方法（object1 == object2）。

 为什么需要重写equals和hashCode方法？
   在我们的业务系统中判断对象时有时候需要的不是一种严格意义上的相等，而是一种业务上的对象相等。在这种情况下，原生的equals方法就不能满足我们的需求了。所以这个时候我们需要重写equals方法，来满足我们的业务系统上的需求。那么为什么在重写equals方法的时候需要重写hashCode方法呢？

   我们先来看一下Object.hashCode的通用约定（摘自《Effective Java》第45页）

1. 在一个应用程序执行期间，如果一个对象的equals方法做比较所用到的信息没有被修改的话，那么，对该对象调用hashCode方法多次，它必须始终如一地返回 同一个整数。在同一个应用程序的多次执行过程中，这个整数可以不同，即这个应用程序这次执行返回的整数与下一次执行返回的整数可以不一致。
2. 如果两个对象根据equals(Object)方法是相等的，那么调用这两个对象中任一个对象的hashCode方法必须产生同样的整数结果。
3. 如果两个对象根据equals(Object)方法是不相等的，那么调用这两个对象中任一个对象的hashCode方法，不要求必须产生不同的整数结果。然而，程序员应该意识到这样的事实，对于不相等的对象产生截然不同的整数结果，有可能提高散列表（hash table）的性能。

**如果只重写了equals方法而没有重写hashCode方法的话，则会违反约定的第二条：相等的对象必须具有相等的散列码（hashCode）。**

   **同时对于HashSet和HashMap这些基于散列值（hash）实现的类。**HashMap的底层处理机制是以数组的方法保存放入的数据的(Node<K,V>[] table)，其中的关键是数组下标的处理。**数组的下标是根据传入的元素hashCode方法的返回值再和特定的值异或决定的。**如果该数组位置上已经有放入的值了，且传入的键值相等则不处理，若不相等则覆盖原来的值，如果数组位置没有条目，则插入，并加入到相应的链表中。检查键是否存在也是根据hashCode值来确定的。所以如果不重写hashCode的话，**可能导致HashSet、HashMap不能正常的运作、**

  如果我们将某个自定义对象存到HashMap或者HashSet及其类似实现类中的时候，**如果该对象的属性参与了hashCode的计算，那么就不能修改该对象参数hashCode计算的属性了。**有可能会移除不了元素，导致内存泄漏。

### JAVA对象的生命周期

对象的整个生命周期大致可以分为7个阶段：创建阶段（Creation）、应用阶段（Using）、不可视阶段（Invisible）、不可到达阶段（Unreachable）、可收集阶段（Collected）、终结阶段（Finalized）与释放阶段（Free）。

### java类生命周期

一个java类的完整的生命周期会经历加载、连接、初始化、使用、和卸载五个阶段

## JAVA数组

### 如何判断一个数组是否有重复数据

1. 若有限定：数组array有 n 个元素，而且元素的取值范围是 1~ n，所以可以通过遍历数组，假设下标 i 位的数字为 j。那么通过交换，将“下标 i 位上的数字 j“ 交换到“下标 j - 1 位上“，直到所有数字都出现在自己对应的下标处，或发生了冲突。此算法的时间复杂度为O(n)，空间复杂度为O(1)。

2.  Hash法 或者 位图法，时间复杂度 和 空间复杂度 都是 O(n)。 

   ————————————————
   版权声明：本文为CSDN博主「小僧_」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
   原文链接：https://blog.csdn.net/cyuyanenen/article/details/51766433

## JAVA泛型

### 泛型擦除，java伪泛型 

[转载](https://www.cnblogs.com/wuqinglong/p/9456193.html)

#### 1. Java泛型的实现方法：类型擦除

大家都知道，Java的泛型是伪泛型，这是因为Java**在编译期间，所有的泛型信息都会被擦掉**，正确理解泛型概念的首要前提是理解类型擦除。**Java的泛型基本上都是在编译器这个层次上实现的，在生成的字节码中是不包含泛型中的类型信息的，使用泛型的时候加上类型参数，在编译器编译的时候会去掉，这个过程成为类型擦除**。

如在代码中定义`List<Object>`和`List<String>`等类型，在编译后都会变成`List`，JVM看到的只是List，而由**泛型附加的类型信息对JVM是看不到的**。Java编译器会在编译时尽可能的发现可能出错的地方，但是仍然无法在运行时刻出现的类型转换异常的情况，**类型擦除也是Java的泛型与C++模板机制实现方式之间的重要区别**。

##### 1-2.通过两个例子证明Java类型的类型擦除

例1.原始类型相等

```java
public class Test {
	public static void main(String[] args) {
        ArrayList<String> list1 = new ArrayList<String>();
        list1.add("abc");
        ArrayList<Integer> list2 = new ArrayList<Integer>();
        list2.add(123);

        System.out.println(list1.getClass() == list2.getClass());
    }
}
```

在这个例子中，我们定义了两个`ArrayList`数组，不过一个是`ArrayList<String>`泛型类型的，只能存储字符串；一个是`ArrayList<Integer>`泛型类型的，只能存储整数，最后，我们通过`list1`对象和`list2`对象的`getClass()`方法获取他们的类的信息，最后发现结果为`true`。说明泛型类型`String`和`Integer`都被擦除掉了，只剩下原始类型。

例2.通过反射添加其它类型元素

```java
public class Test {

    public static void main(String[] args) throws Exception {
        ArrayList<Integer> list = new ArrayList<Integer>();
        list.add(1);  //这样调用 add 方法只能存储整形，因为泛型类型的实例为 Integer
        list.getClass().getMethod("add", Object.class).invoke(list, "asd");
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
}
```

在程序中定义了一个`ArrayList<Integer>`泛型类型实例化为`Integer`对象，如果直接调用`add()`方法，那么只能存储整数数据，不过当我们利用反射调用`add()`方法的时候，却可以存储字符串，这说明了`Integer`泛型实例在编译之后被擦除掉了，只保留了原始类型。

#### 2.类型擦除后保留的原始类型

在上面，两次提到了原始类型，什么是原始类型？

**原始类型** 就是擦除去了泛型信息，最后在字节码中的类型变量的真正类型，无论何时定义一个泛型，相应的原始类型都会被自动提供，类型变量擦除，并使用其限定类型（无限定的变量用Object）替换。

例3.原始类型Object

```java
class Pair<T> {  
    private T value;  
    public T getValue() {  
        return value;  
    }  
    public void setValue(T  value) {  
        this.value = value;  
    }  
}  
```

Pair的原始类型为:

```java
class Pair {  
    private Object value;  
    public Object getValue() {  
        return value;  
    }  
    public void setValue(Object  value) {  
        this.value = value;  
    }  
}
```

因为在`Pair<T>`中，T 是一个无限定的类型变量，所以用`Object`替换，其结果就是一个普通的类，如同泛型加入Java语言之前的已经实现的样子。在程序中可以包含不同类型的`Pair`，如`Pair<Integer>`或`Pair<String>`，但是擦除类型后他们的就成为原始的`Pair`类型了，原始类型都是`Object`。

从上面的例2中，我们也可以明白`ArrayList`被擦除类型后，原始类型也变为`Object`，所以通过反射我们就可以存储字符串了。

如果类型变量有限定，那么原始类型就用第一个边界的类型变量类替换。

比如: Pair这样声明的话

```java
public class Pair<T extends Comparable> {}
```

那么原始类型就是`Comparable`。

要区分原始类型和泛型变量的类型。

**在调用泛型方法时，可以指定泛型，也可以不指定泛型。**

- 在不指定泛型的情况下，泛型变量的类型为该方法中的几种类型的同一父类的最小级，直到Object
- 在指定泛型的情况下，该方法的几种类型必须是该泛型的实例的类型或者其子类

```java
public class Test {  
    public static void main(String[] args) {  

        /**不指定泛型的时候*/  
        int i = Test.add(1, 2); //这两个参数都是Integer，所以T为Integer类型  
        Number f = Test.add(1, 1.2); //这两个参数一个是Integer，以风格是Float，所以取同一父类的最小级，为Number  
        Object o = Test.add(1, "asd"); //这两个参数一个是Integer，以风格是Float，所以取同一父类的最小级，为Object  

        /**指定泛型的时候*/  
        int a = Test.<Integer>add(1, 2); //指定了Integer，所以只能为Integer类型或者其子类  
        int b = Test.<Integer>add(1, 2.2); //编译错误，指定了Integer，不能为Float  
        Number c = Test.<Number>add(1, 2.2); //指定为Number，所以可以为Integer和Float  
    }  

    //这是一个简单的泛型方法  
    public static <T> T add(T x,T y){  
        return y;  
    }  
}
```

其实在泛型类中，不指定泛型的时候，也差不多，只不过这个时候的泛型为`Object`，就比如`ArrayList`中，如果不指定泛型，那么这个`ArrayList`可以存储任意的对象。

例4.Object泛型

```java
public static void main(String[] args) {  
    ArrayList list = new ArrayList();  
    list.add(1);  
    list.add("121");  
    list.add(new Date());  
}  
```

#### 3.类型擦除引起的问题及解决方法

因为种种原因，**Java不能实现真正的泛型，只能使用类型擦除来实现伪泛型，这样虽然不会有类型膨胀问题，但是也引起来许多新问题，所以，SUN对这些问题做出了种种限制，避免我们发生各种错误。**

##### 3-1.先检查，再编译以及编译的对象和引用传递问题

Q: 既然说类型变量会在编译的时候擦除掉，那为什么我们往 ArrayList 创建的对象中添加整数会报错呢？不是说泛型变量String会在编译的时候变为Object类型吗？为什么不能存别的类型呢？既然类型擦除了，如何保证我们只能使用泛型变量限定的类型呢？

A: Java编译器是通过先检查代码中泛型的类型，然后在进行类型擦除，再进行编译。

例如：

```java
public static  void main(String[] args) {  

    ArrayList<String> list = new ArrayList<String>();  
    list.add("123");  
    list.add(123);//编译错误  
}
```

在上面的程序中，使用`add`方法添加一个整型，在IDE中，直接会报错，说明这就是在编译之前的检查，因为如果是在编译之后检查，类型擦除后，原始类型为`Object`，是应该允许任意引用类型添加的。可实际上却不是这样的，这恰恰说明了关于泛型变量的使用，是会在编译之前检查的。

那么，这个类型检查是针对谁的呢？我们先看看参数化类型和原始类型的兼容。

以 ArrayList举例子，以前的写法:

```java
ArrayList list = new ArrayList();  
```

现在的写法:

```java
ArrayList<String> list = new ArrayList<String>();
```

如果是与以前的代码兼容，各种引用传值之间，必然会出现如下的情况：

```java
ArrayList<String> list1 = new ArrayList(); //第一种 情况
ArrayList list2 = new ArrayList<String>(); //第二种 情况
```

**这样是没有错误的，不过会有个编译时警告。**

不过在第一种情况，可以实现与完全使用泛型参数一样的效果，第二种则没有效果。

因为类型检查就是编译时完成的，`new ArrayList()`只是在内存中开辟了一个存储空间，可以存储任何类型对象，而**真正设计类型检查的是它的引用**，因为我们是使用它引用`list1`来调用它的方法，比如说调用`add`方法，所以`list1`引用能完成泛型类型的检查。而引用`list2`没有使用泛型，所以不行。

举例子：

```java
public class Test {  
    public static void main(String[] args) {  

        ArrayList<String> list1 = new ArrayList();  
        list1.add("1"); //编译通过  
        list1.add(1); //编译错误  
        String str1 = list1.get(0); //返回类型就是String  

        ArrayList list2 = new ArrayList<String>();  
        list2.add("1"); //编译通过  
        list2.add(1); //编译通过  
        Object object = list2.get(0); //返回类型就是Object  

        new ArrayList<String>().add("11"); //编译通过  
        new ArrayList<String>().add(22); //编译错误  

        String str2 = new ArrayList<String>().get(0); //返回类型就是String  
    }  

}  
```

通过上面的例子，我们可以明白，类型检查就是针对引用的，谁是一个引用，用这个引用调用泛型方法，就会对这个引用调用的方法进行类型检测，而无关它真正引用的对象。

泛型中参数话类型为什么不考虑继承关系？

在Java中，像下面形式的引用传递是不允许的:

```java
ArrayList<String> list1 = new ArrayList<Object>(); //编译错误  
ArrayList<Object> list2 = new ArrayList<String>(); //编译错误
```

我们先看第一种情况，将第一种情况拓展成下面的形式：

```java
ArrayList<Object> list1 = new ArrayList<Object>();  
list1.add(new Object());  
list1.add(new Object());  
ArrayList<String> list2 = list1; //编译错误
```

实际上，在第4行代码的时候，就会有编译错误。那么，我们先假设它编译没错。那么当我们使用`list2`引用用`get()`方法取值的时候，返回的都是`String`类型的对象（上面提到了，类型检测是根据引用来决定的），可是它里面实际上已经被我们存放了`Object`类型的对象，这样就会有`ClassCastException`了。所以为了避免这种极易出现的错误，Java不允许进行这样的引用传递。（这也是泛型出现的原因，就是为了解决类型转换的问题，我们不能违背它的初衷）。

再看第二种情况，将第二种情况拓展成下面的形式：

```java
ArrayList<String> list1 = new ArrayList<String>();  
list1.add(new String());  
list1.add(new String());

ArrayList<Object> list2 = list1; //编译错误
```

没错，这样的情况比第一种情况好的多，最起码，在我们用`list2`取值的时候不会出现`ClassCastException`，因为是从`String`转换为`Object`。可是，这样做有什么意义呢，泛型出现的原因，就是为了解决类型转换的问题。我们使用了泛型，到头来，还是要自己强转，违背了泛型设计的初衷。所以java不允许这么干。再说，你如果又用`list2`往里面`add()`新的对象，那么到时候取得时候，我怎么知道我取出来的到底是`String`类型的，还是`Object`类型的呢？

**所以，要格外注意，泛型中的引用传递的问题。**

##### 3-2.自动类型转换

因为类型擦除的问题，所以所有的泛型类型变量最后都会被替换为原始类型。既然都被替换为原始类型，那么为什么我们在获取的时候，不需要进行强制类型转换呢？

看下`ArrayList.get()`方法：

```java
public E get(int index) {  
    RangeCheck(index); 
    return (E) elementData[index];  
}
```

可以看到，在`return`之前，会根据泛型变量进行强转。假设泛型类型变量为`Date`，虽然泛型信息会被擦除掉，但是会将`(E) elementData[index]`，编译为`(Date)elementData[index]`。所以我们不用自己进行强转。当存取一个泛型域时也会自动插入强制类型转换。假设`Pair`类的`value`域是`public`的，那么表达式：

```java
Date date = pair.value;
```

也会自动地在结果字节码中插入强制类型转换。

##### 3-3.类型擦除与多态的冲突和解决方法

现在有这样一个泛型类：

```java
class Pair<T> {  
    private T value;  
    public T getValue() {  
        return value;  
    }  
    public void setValue(T value) {  
        this.value = value;  
    }  
}
```

然后我们想要一个子类继承它。

```java
class DateInter extends Pair<Date> {  
    @Override  
    public void setValue(Date value) {  
        super.setValue(value);  
    }  
    @Override  
    public Date getValue() {  
        return super.getValue();  
    }  
}
```

在这个子类中，我们设定父类的泛型类型为`Pair`，在子类中，我们覆盖了父类的两个方法，我们的原意是这样的：将父类的泛型类型限定为`Date`，那么父类里面的两个方法的参数都为`Date`类型。

```java
public Date getValue() {  
    return value;  
}  

public void setValue(Date value) {  
    this.value = value;  
}
```

所以，我们在子类中重写这两个方法一点问题也没有，实际上，从他们的`@Override`标签中也可以看到，一点问题也没有，实际上是这样的吗？

分析：实际上，类型擦除后，父类的的泛型类型全部变为了原始类型`Object`，所以父类编译之后会变成下面的样子：

```java
class Pair {  
    private Object value;  

    public Object getValue() {  
        return value;  
    }  

    public void setValue(Object  value) {  
        this.value = value;  
    }  
}  
```

再看子类的两个重写的方法的类型：

```java
@Override  
public void setValue(Date value) {  
    super.setValue(value);  
}  
@Override  
public Date getValue() {  
    return super.getValue();  
}
```

先来分析`setValue`方法，父类的类型是`Object`，而子类的类型是`Date`，参数类型不一样，这如果实在普通的继承关系中，根本就不会是重写，而是重载。
　　
我们在一个main方法测试一下：

```java
public static void main(String[] args) throws ClassNotFoundException {  
        DateInter dateInter = new DateInter();  
        dateInter.setValue(new Date());                  
        dateInter.setValue(new Object()); //编译错误  
}
```

如果是重载，那么子类中两个`setValue`方法，一个是参数`Object`类型，一个是`Date`类型，可是我们发现，根本就没有这样的一个子类继承自父类的Object类型参数的方法。所以说，却是是重写了，而不是重载了。

为什么会这样呢？

原因是这样的，我们传入父类的泛型类型是`Date，Pair`，我们的本意是将泛型类变为如下：

```java
class Pair {  
    private Date value;  
    public Date getValue() {  
        return value;  
    }  
    public void setValue(Date value) {  
        this.value = value;  
    }  
}
```

然后再子类中重写参数类型为Date的那两个方法，实现继承中的多态。

可是由于种种原因，**虚拟机并不能将泛型类型变为`Date`，只能将类型擦除掉，变为原始类型`Object`。这样，我们的本意是进行重写，实现多态。可是类型擦除后，只能变为了重载。**这样，类型擦除就和多态有了冲突。JVM知道你的本意吗？知道！！！可是它能直接实现吗，不能！！！如果真的不能的话，那我们怎么去重写我们想要的`Date`类型参数的方法啊。

于是**JVM采用了一个特殊的方法，来完成这项功能，那就是桥方法**。

首先，我们用`javap -c className`的方式反编译下`DateInter`子类的字节码，结果如下：

```class
class com.tao.test.DateInter extends com.tao.test.Pair<java.util.Date> {  
  com.tao.test.DateInter();  
    Code:  
       0: aload_0  
       1: invokespecial #8                  // Method com/tao/test/Pair."<init>":()V  
       4: return  

  public void setValue(java.util.Date);  //我们重写的setValue方法  
    Code:  
       0: aload_0  
       1: aload_1  
       2: invokespecial #16                 // Method com/tao/test/Pair.setValue:(Ljava/lang/Object;)V  
       5: return  

  public java.util.Date getValue();    //我们重写的getValue方法  
    Code:  
       0: aload_0  
       1: invokespecial #23                 // Method com/tao/test/Pair.getValue:()Ljava/lang/Object;  
       4: checkcast     #26                 // class java/util/Date  
       7: areturn  

  public java.lang.Object getValue();     //编译时由编译器生成的巧方法  
    Code:  
       0: aload_0  
       1: invokevirtual #28                 // Method getValue:()Ljava/util/Date 去调用我们重写的getValue方法;  
       4: areturn  

  public void setValue(java.lang.Object);   //编译时由编译器生成的巧方法  
    Code:  
       0: aload_0  
       1: aload_1  
       2: checkcast     #26                 // class java/util/Date  
       5: invokevirtual #30                 // Method setValue:(Ljava/util/Date; 去调用我们重写的setValue方法)V  
       8: return  
}
```

从编译的结果来看，我们本意重写`setValue`和`getValue`方法的子类，竟然有4个方法，其实不用惊奇，最后的两个方法，就是编译器自己生成的桥方法。可以看到桥方法的参数类型都是Object，也就是说，子类中真正覆盖父类两个方法的就是这两个我们看不到的桥方法。而打在我们自己定义的`setvalue`和`getValue`方法上面的`@Oveerride`只不过是假象。而桥方法的内部实现，就只是去调用我们自己重写的那两个方法。

所以，虚拟机巧妙的使用了桥方法，来解决了类型擦除和多态的冲突。

不过，要提到一点，这里面的`setValue`和`getValue`这两个桥方法的意义又有不同。

`setValue`方法是为了解决类型擦除与多态之间的冲突。

而`getValue`却有普遍的意义，怎么说呢，如果这是一个普通的继承关系：

那么父类的`setValue`方法如下：

```java
public ObjectgetValue() {  
    return super.getValue();  
}
```

而子类重写的方法是：

```java
public Date getValue() {  
    return super.getValue();  
}
```

其实这在普通的类继承中也是普遍存在的重写，这就是协变。

关于协变：。。。。。。

并且，还有一点也许会有疑问，子类中的巧方法`Object getValue()`和`Date getValue()`是同 时存在的，可是如果是常规的两个方法，他们的方法签名是一样的，也就是说虚拟机根本不能分别这两个方法。如果是我们自己编写Java代码，这样的代码是无法通过编译器的检查的，但是虚拟机却是允许这样做的，因为虚拟机通过参数类型和返回类型来确定一个方法，所以编译器为了实现泛型的多态允许自己做这个看起来“不合法”的事情，然后交给虚拟器去区别。

##### 3-4.泛型类型变量不能是基本数据类型

不能用类型参数替换基本类型。就比如，没有`ArrayList`，只有`ArrayList`。因为当类型擦除后，`ArrayList`的原始类型变为`Object`，但是`Object`类型不能存储`double`值，只能引用`Double`的值。

##### 3-5.编译时集合的instanceof

```java
ArrayList<String> arrayList = new ArrayList<String>();
```

因为类型擦除之后，`ArrayList`只剩下原始类型，泛型信息`String`不存在了。

那么，编译时进行类型查询的时候使用下面的方法是错误的

```java
if( arrayList instanceof ArrayList<String>)
```

##### 3-6.泛型在静态方法和静态类中的问题

泛型类中的静态方法和静态变量不可以使用泛型类所声明的泛型类型参数

举例说明：

```java
public class Test2<T> {    
    public static T one;   //编译错误    
    public static  T show(T one){ //编译错误    
        return null;    
    }    
}
```

因为泛型类中的**泛型参数的实例化是在定义对象的时候指定的，而静态变量和静态方法不需要使用对象来调用。对象都没有创建，如何确定这个泛型参数是何种类型，所以当然是错误的。**

但是要注意区分下面的一种情况：

```java
public class Test2<T> {    

    public static <T >T show(T one){ //这是正确的    
        return null;    
    }    
}
```

因为这是一个泛型方法，在泛型方法中使用的T是自己在方法中定义的 T，而不是泛型类中的T。

## 反射

## IO-BIO、NIO、AIO

**同步与异步**

- **同步：** 同步就是发起一个调用后，被调用者未处理完请求之前，调用不返回。
- **异步：** 异步就是发起一个调用后，立刻得到被调用者的回应表示已接收到请求，但是被调用者并没有返回结果，此时我们可以处理其他的请求，被调用者通常依靠事件，回调等机制来通知调用者其返回结果。

同步和异步的区别最大在于异步的话调用者不需要等待处理结果，被调用者会通过回调等机制来通知调用者其返回结果。

**阻塞和非阻塞**

- **阻塞：** 阻塞就是发起一个请求，调用者一直等待请求结果返回，也就是当前线程会被挂起，无法从事其他任务，只有当条件就绪才能继续。
- **非阻塞：** 非阻塞就是发起一个请求，调用者不用一直等着结果返回，可以先去干其他事情。

### **BIO**：同步阻塞式IO

采用 **BIO 通信模型** 的服务端，通常由一个独立的 **Acceptor 线程**负责监听**客户端的连接**。我们一般通过在 `while(true)` 循环中服务端会调用 `accept()` 方法等待接收客户端的连接的方式监听请求，请求一旦接收到一个连接请求，就可以建立通信套接字在这个通信套接字上进行读写操作，此时不能再接收其他客户端连接请求，只能等待同当前连接的客户端的操作执行完成， 不过可以通过多线程来支持多个客户端的连接。

如果要让 **BIO 通信模型** 能够同时处理多个客户端请求，就必须使用多线程**（主要原因是 `socket.accept()`、 `socket.read()`、 `socket.write()` 涉及的三个主要函数都是同步阻塞的）**，也就是说它在接收到客户端连接请求之后为每个客户端创建一个新的线程进行链路处理，处理完成之后，通过输出流返回应答给客户端，线程销毁。这就是典型的 **一请求一应答通信模型** 。我们可以设想一下如果这个连接不做任何事情的话就会造成不必要的线程开销，不过可以通过 **线程池机制** 改善，线程池还可以让线程的创建和回收成本相对较低。使用`FixedThreadPool` 可以有效的控制了线程的最大数量，保证了系统有限的资源的控制，实现了N(客户端请求数量):M(处理客户端请求的线程数量)的伪异步I/O模型（N 可以远远大于 M）。

**我们再设想一下当客户端并发访问量增加后这种模型会出现什么问题？**

在 Java 虚拟机中，线程是宝贵的资源，线程的创建和销毁成本很高，除此之外，线程的切换成本也是很高的。尤其在 Linux 这样的操作系统中，线程本质上就是一个进程，创建和销毁线程都是重量级的系统函数。如果并发访问量增加会导致线程数急剧膨胀可能会导致线程堆栈溢出、创建新线程失败等问题，最终导致进程宕机或者僵死，不能对外提供服务。

### **NIO**：同步非阻塞式IO

NIO是一种同步非阻塞的I/O模型，在Java 1.4 中引入了NIO框架，对应 java.nio 包，提供了 Channel , Selector，Buffer等抽象。

NIO中的N可以理解为Non-blocking，不单纯是New。它支持面向缓冲的，基于通道的I/O操作方法。 NIO提供了与传统BIO模型中的 `Socket` 和 `ServerSocket` 相对应的 `SocketChannel` 和 `ServerSocketChannel` 两种不同的套接字通道实现,两种通道都支持阻塞和非阻塞两种模式。阻塞模式使用就像传统中的支持一样，比较简单，但是性能和可靠性都不好；非阻塞模式正好与之相反。对于低负载、低并发的应用程序，可以使用同步阻塞I/O来提升开发速率和更好的维护性；对于高负载、高并发的（网络）应用，应使用 NIO 的非阻塞模式来开发。

**NIO 的主要组成部分**：

1）**Buffer**，高效的数据容器，除了布尔类型，所有原始数据类型都有相应的 Buffer 实现。

2）**Channel，**类似在 Linux 之类操作系统上看到的文件描述符，是 NIO 中被用来支持批量式 IO 操作的一种抽象。

3）**Selector**，是 **NIO 实现多路复用的基础**，它提供了一种高效的机制，可以**检测到注册在 Selector 上的多个 Channel 中，是否有 Channel 处于就绪状态，进而实现了单线程对多 Channel 的高效管理**（Selector 基于底层操作系统机制，Linux 上依赖于 epoll、Windows上NIO2（AIO）模式则是依赖于iocp）

4）**Chartset**，提供 Unicode 字符串定义

所以NIO是如何实现多路复用的？：

1）首先，通过 Selector.open() 创建一个 Selector，作为类似调度员的角色。

2）然后，创建一个 ServerSocketChannel，并且向 Selector 注册，通过指定 SelectionKey.OP_ACCEPT，告诉调度员，它关注的是新的连接请求。注意，为什么我们要明确配置非阻塞模式呢？这是因为阻塞模式下，注册操作是不允许的，会抛出 IllegalBlockingModeException 异常。

3）Selector 阻塞在 select 操作，当有 Channel 发生接入请求，就会被唤醒。

NIO 利用了单线程轮询事件的机制，通过高效地定位就绪的 Channel，来决定做什么，仅仅 select 阶段是阻塞的，可以有效避免大量客户端连接时，频繁线程切换带来的问题，应用的扩展能力有了非常大的提高。

- JDK 的 NIO 底层由 epoll 实现，该实现饱受诟病的空轮询 bug 会导致 cpu 飙升 100%
- 项目庞大之后，自行实现的 NIO 很容易出现各类 bug，维护成本较高

### **AIO**：异步非阻塞式IO

异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。

AIO 是异步IO的缩写，虽然 NIO 在网络操作中，提供了非阻塞的方法，但是 NIO 的 IO 行为还是同步的。对于 NIO 来说，我们的业务线程是在 IO 操作准备好时，得到通知，接着就由这个线程自行进行 IO 操作，IO操作本身是同步的。

### 操作系统IO模型

* 操作系统启动后内存中第一个启动的进程为kernel（内核），负责管理所有硬件资源。
* 内核启动后会创建一个GDT（全局描述符），GDT将内存空间划分为**用户空间**和**内核空间**以保护操作系统内核。程序是无法直接访问硬件是，只能通过系统调用访问内核空间。
* 系统调用（System call）会产生一个软中断，CPU被软中断后会根据中断号找到内核中对应的callback函数，通过内核空间的callback访问硬件资源。

### Netty

NIO包装成AIO的框架

### Epoll（I/O Event Poll）

在Linux的Man文档中，我们可以看到如下定义

> Epoll - I/O event notification facility
>

翻译一下，epoll是一种I/O事件通知机制，这句话基本上包含了所有需要理解的要点。

* **I/O事件**
  * 基于file descriptor，支持file, socket, pipe等各种I/O方式
  * 当文件描述符关联的内核读缓冲区可读，则触发可读事件，什么是可读呢？就是内核缓冲区非空，有数据可以读取
  * 当文件描述符关联的内核写缓冲区可写，则触发可写事件，什么是可写呢？就是内核缓冲区不满，有空闲空间可以写入
* 通知机制
  * 通知机制，就是当事件发生的时候，去通知他
  * 通知机制的反面，就是轮询机制

epoll是一种当文件描述符的内核缓冲区非空的时候，发出可读信号进行通知，当写缓冲区不满的时候，发出可写信号通知的机制

**水平触发与边缘触发**

* **水平触发(level-trggered)**

  只要文件描述符关联的读内核缓冲区非空，有数据可以读取，就一直发出可读信号进行通知，当文件描述符关联的内核写缓冲区不满，有空间可以写入，就一直发出可写信号进行通知。

* **边缘触发(edge-triggered)**
  当文件描述符关联的读内核缓冲区由空转化为非空的时候，则发出可读信号进行通知，当文件描述符关联的内核写缓冲区由满转化为不满的时候，则发出可写信号进行通知。
  所以边缘触发需要一次性的把缓冲区的数据读完为止，也就是一直读，直到读到EGAIN为止，EGAIN说明缓冲区已经空了，因为这一点，边缘触发需要设置文件句柄为非阻塞

### select、poll、epoll之间的区别

(1)select==>时间复杂度O(n)

它仅仅知道了，有I/O事件发生了，却并不知道是哪那几个流（可能有一个，多个，甚至全部），我们只能无差别轮询所有流，找出能读出数据，或者写入数据的流，对他们进行操作。所以**select具有O(n)的无差别轮询复杂度**，同时处理的流越多，无差别轮询时间就越长。

(2)poll==>时间复杂度O(n)

poll本质上和select没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个fd对应的设备状态， **但是它没有最大连接数的限制**，原因是它是基于链表来存储的.

(3)epoll==>时间复杂度O(1)

**epoll可以理解为event poll**，不同于忙轮询和无差别轮询，epoll会把哪个流发生了怎样的I/O事件通知我们。所以我们说epoll实际上是**事件驱动（每个事件关联上fd）**的，此时我们对这些流的操作都是有意义的。**（复杂度降低到了O(1)）**

select，poll，epoll都是IO多路复用的机制。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。**但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的**，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。 

epoll跟select都能提供多路I/O复用的解决方案。在现在的Linux内核里有都能够支持，其中epoll是Linux所特有，而select则应该是POSIX所规定，一般操作系统均有实现。

### RPC

RPC即远程过程调用，在客户端远程调用服务端的一个方法。

**核心：动态代理，序列化**

RPC要达到的目标：远程调用时，要能够像本地调用一样方便，让调用者感知不到远程调用的逻辑。

**实现方法**：RPC 服务端通过RpcServer去导出（export）远程接口方法，而客户端通过RpcClient去导入（import）远程接口方法。客户端像调用本地方法一样去调用远程接口方法，**RPC 框架提供接口的代理实现**，实际的调用将委托给代理RpcProxy。代理封装调用信息并将调用转交给RpcInvoker去实际执行。在客户端的RpcInvoker通过连接器RpcConnector去维持与服务端的通道RpcChannel，并使用RpcProtocol执行协议编码（encode）并将编码后的请求消息通过通道发送给服务端。
RPC 服务端接收器RpcAcceptor接收客户端的调用请求，同样使用RpcProtocol执行协议解码（decode）。
解码后的调用信息传递给RpcProcessor去控制处理调用过程，最后再委托调用给RpcInvoker去实际执行并返回调用结果。

**实现RPC需要解决的问题：**

- **Call ID映射**。我们怎么告诉远程机器我们要**调用哪个函数呢**？在本地调用中，函数体是直接通过函数指针来指定的，我们调用具体函数，编译器就自动帮我们调用它相应的函数指针。但是在远程调用中，是无法调用函数指针的，因为两个进程的地址空间是完全不一样。所以，在RPC中，**所有的函数都必须有自己的一个ID**。这个ID在所有进程中都是唯一确定的。客户端在做远程过程调用时，必须附上这个ID。然后我们还需要在客户端和服务端分别维护一个 {函数 <--> Call ID} 的对应表。两者的表不一定需要完全相同，但相同的函数对应的Call ID必须相同。当客户端需要进行远程调用时，它就查一下这个表，找出相应的Call ID，然后把它传给服务端，服务端也通过查表，来确定客户端需要调用的函数，然后执行相应函数的代码。
- **序列化和反序列化**。客户端怎么把参数值传给远程的函数呢？在本地调用中，我们只需要把参数压到栈里，然后让函数自己去栈里读就行。但是在远程过程调用时，客户端跟服务端是不同的进程，**不能通过内存来传递参数**。甚至有时候客户端和服务端使用的都**不是同一种语言**（比如服务端用C++，客户端用Java或者Python）。这时候就需要客户端把参数先转成一个字节流，传给服务端后，再把字节流转成自己能读取的格式。这个过程叫序列化和反序列化。同理，从服务端返回的值也需要序列化反序列化的过程。
- **网络传输**。远程调用往往是基于网络的，客户端和服务端是通过网络连接的。所有的数据都需要通过网络传输，因此就需要有一个网络传输层。网络传输层需要把Call ID和序列化后的参数字节流传给服务端，然后再把序列化后的调用结果传回客户端。只要能完成这两者的，都可以作为传输层使用。因此，它所使用的协议其实是不限的，能完成传输就行。尽管大部分RPC框架都使用TCP协议，但其实UDP也可以，而gRPC干脆就用了HTTP2。Java的Netty也属于这层的东西。

 **实现高可用RPC框架需要考虑到的问题**：

- 既然系统采用分布式架构，那一个服务势必会有多个实例，要解决**如何获取实例的问题**。所以需要一个服务注册中心，比如在Dubbo中，就可以使用Zookeeper作为注册中心，在调用时，从Zookeeper获取服务的实例列表，再从中选择一个进行调用；
- 如何选择实例呢？就要考虑负载均衡，例如dubbo提供了4种负载均衡策略；
- 如果每次都去注册中心查询列表，效率很低，那么就要加缓存；
- 客户端总不能每次调用完都等着服务端返回数据，所以就要支持异步调用；
- 服务端的接口修改了，老的接口还有人在用，这就需要版本控制；
- 服务端总不能每次接到请求都马上启动一个线程去处理，于是就需要线程池；



## 其他

### Protoco Buffer 

Protocol Buffer 是 Google 出品的一种轻量 & 高效的结构化数据存储格式，性能比 Json、XML 真的强！太！多！

**Protocol Buffer 的序列化 & 反序列化简单 & 速度快的原因是：**

1. 编码 / 解码 方式简单（只需要简单的数学运算 = 位移等等）
2. 采用 Protocol Buffer 自身的框架代码 和 编译器 共同完成

**Protocol Buffer 的数据压缩效果好（即序列化后的数据量体积小）的原因是：**

1. 采用了独特的编码方式，如 Varint、Zigzag 编码方式等等
2. 采用 T - L - V 的数据存储方式：减少了分隔符的使用 & 数据存储得紧凑

### 面向对象/类的四个特性

java面向对象编程是指：Java语言提供类、接口和继承等面向对象的特性，为了简单起见，只支持类之间的单继承，但支持接口之间的多继承，并支持类与接口之间的实现机制（关键字为implements）。另外，java是面向对象，但不是所有的都是对象，基本数据类型就不是对象，所以才会有封装类。

面向对象的特性有1）抽象，2）封装，3）继承，4）多态。

### 字节码技术、动态代理、字节增强



