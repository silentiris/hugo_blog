+++
title = "Java注解原理"
date = "2023-06-01T21:37:02+08:00"
tags = []
slug = "Java-annotation-principle"
+++

# 注解的基础知识

注解是JDK1.5版本开始引入的一个特性，用于对代码进行说明，可以对包、类、接口、字段、方法参数、局部变量等进行注解。它主要的作用有以下四方面：

-   生成文档，通过代码里标识的元数据生成javadoc文档。
-   编译检查，通过代码里标识的元数据让编译器在编译期间进行检查验证。
-   编译时动态处理，编译时通过代码里标识的元数据动态处理，例如动态生成代码。
-   运行时动态处理，运行时通过代码里标识的元数据动态处理，例如使用反射注入实例。

这么来说是比较抽象的，我们具体看下注解的常见分类：

-   **Java自带的标准注解**，包括`@Override`、`@Deprecated`和`@SuppressWarnings`，分别用于标明重写某个方法、标明某个类或方法过时、标明要忽略的警告，用这些注解标明后编译器就会进行检查。
-   **元注解**，元注解是用于定义注解的注解，包括`@Retention`、`@Target`、`@Inherited`、`@Documented`，`@Retention`用于标明注解被保留的阶段，`@Target`用于标明注解使用的范围，`@Inherited`用于标明注解可继承，`@Documented`用于标明是否生成javadoc文档。
-   **自定义注解**，可以根据自己的需求定义注解，并可用元注解对自定义注解进行注解。

## java内置注解

Java 1.5开始自带的标准注解，包括`@Override`、`@Deprecated`和`@SuppressWarnings`：

-   `@Override`：表示当前的方法定义将覆盖父类中的方法

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

-   `@Deprecated`：表示代码被弃用，如果使用了被@Deprecated注解的代码则编译器将发出警告

```java
@Documented @Retention(RetentionPolicy.RUNTIME)
@Target(value={CONSTRUCTOR, FIELD, LOCAL_VARIABLE, METHOD, PACKAGE, PARAMETER, TYPE}) public @interface Deprecated { 
}
```

-   `@SuppressWarnings`：表示关闭编译器警告信息

```java
@Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE}) @Retention(RetentionPolicy.SOURCE) 
public @interface SuppressWarnings { 
	String[] value();
 }
```

## 元注解

上述内置注解的定义中使用了一些元注解（注解类型进行注解的注解类），在JDK 1.5中提供了4个标准的元注解：`@Target`，`@Retention`，`@Documented`，`@Inherited`, 在JDK 1.8中提供了两个元注解 `@Repeatable`和`@Native`。

### 元注解 - @Target

> Target注解的作用是：描述注解的使用范围（即：被修饰的注解可以用在什么地方） 。

Target注解用来说明那些被它所注解的注解类可修饰的对象范围：注解可以用于修饰 packages、types（类、接口、枚举、注解类）、类成员（方法、构造方法、成员变量、枚举值）、方法参数和本地变量（如循环变量、catch参数），在定义注解类时使用了@Target 能够更加清晰的知道它能够被用来修饰哪些对象，它的取值范围定义在ElementType 枚举中。

```java
public enum ElementType {
 
    TYPE, // 类、接口、枚举类
 
    FIELD, // 成员变量（包括：枚举常量）
 
    METHOD, // 成员方法
 
    PARAMETER, // 方法参数
 
    CONSTRUCTOR, // 构造方法
 
    LOCAL_VARIABLE, // 局部变量
 
    ANNOTATION_TYPE, // 注解类

    PACKAGE, // 可用于修饰：包
 
    TYPE_PARAMETER, // 类型参数，JDK 1.8 新增
 
    TYPE_USE // 使用类型的任何地方，JDK 1.8 新增
 
}
```

### 元注解 - @Retention & @RetentionTarget

> Reteniton注解的作用是：描述注解保留的时间范围（即：被描述的注解在它所修饰的类中可以被保留到何时） 。

Reteniton注解用来限定那些被它所注解的注解类在注解到其他类上以后，可被保留到何时，一共有三种策略，定义在RetentionPolicy枚举中。

```java
public enum RetentionPolicy {
 
    SOURCE,    // 源文件保留
    CLASS,       // 编译期保留，默认值
    RUNTIME   // 运行期保留，可通过反射去获取注解信息
}
```

### 元注解 - @Documented

> Documented注解的作用是：描述在使用 javadoc 工具为类生成帮助文档时是否要保留其注解信息。

以下代码在使用Javadoc工具可以生成`@TestDocAnnotation`注解信息。

```java
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Target;
 
@Documented
@Target({ElementType.TYPE,ElementType.METHOD})
public @interface TestDocAnnotation {
 
	public String value() default "default";
}
```

#### 元注解 - @Inherited

> Inherited注解的作用：被它修饰的Annotation将具有继承性。如果某个类使用了被@Inherited修饰的Annotation，则其子类将自动具有该注解。

## 注解与反射接口

> 定义注解后，如何获取注解中的内容呢？反射包java.lang.reflect下的AnnotatedElement接口提供这些方法。这里注意：只有注解被定义为RUNTIME后，该注解才能是运行时可见，当class文件被装载时被保存在class文件中的Annotation才会被虚拟机读取。

AnnotatedElement 接口是所有程序元素（Class、Method和Constructor）的父接口，所以程序通过反射获取了某个类的AnnotatedElement对象之后，程序就可以调用该对象的方法来访问Annotation信息。我们看下具体的先关接口

-   `boolean isAnnotationPresent(Class<?extends Annotation> annotationClass)`

判断该程序元素上是否包含指定类型的注解，存在则返回true，否则返回false。注意：此方法会忽略注解对应的注解容器。

-   `<T extends Annotation> T getAnnotation(Class<T> annotationClass)`

返回该程序元素上存在的、指定类型的注解，如果该类型注解不存在，则返回null。

-   `Annotation[] getAnnotations()`

返回该程序元素上存在的所有注解，若没有注解，返回长度为0的数组。

-   `<T extends Annotation> T[] getAnnotationsByType(Class<T> annotationClass)`

返回该程序元素上存在的、指定类型的注解数组。没有注解对应类型的注解时，返回长度为0的数组。该方法的调用者可以随意修改返回的数组，而不会对其他调用者返回的数组产生任何影响。`getAnnotationsByType`方法与 `getAnnotation`的区别在于，`getAnnotationsByType`会检测注解对应的重复注解容器。若程序元素为类，当前类上找不到注解，且该注解为可继承的，则会去父类上检测对应的注解。

-   `<T extends Annotation> T getDeclaredAnnotation(Class<T> annotationClass)`

返回直接存在于此元素上的所有注解。与此接口中的其他方法不同，该方法将忽略继承的注释。如果没有注释直接存在于此元素上，则返回null

-   `<T extends Annotation> T[] getDeclaredAnnotationsByType(Class<T> annotationClass)`

返回直接存在于此元素上的所有注解。与此接口中的其他方法不同，该方法将忽略继承的注释

-   `Annotation[] getDeclaredAnnotations()`

返回直接存在于此元素上的所有注解及注解对应的重复注解容器。与此接口中的其他方法不同，该方法将忽略继承的注解。如果没有注释直接存在于此元素上，则返回长度为零的一个数组。该方法的调用者可以随意修改返回的数组，而不会对其他调用者返回的数组产生任何影响。

## 自定义注解

eg:

```java
@Target(ElementType.METHOD) 
@Retention(RetentionPolicy.RUNTIME) 
public @interface MyMethodAnnotation { 
	public String title() default ""; 
	public String description() default ""; 
}
```

# 前置知识：动态代理

在动态代理中，通过 `Proxy` 类的 `newProxyInstance()` 方法创建代理对象时，会在运行时动态生成一个新的代理类。
具体的代理类生成过程如下：

1.  使用 `Proxy.getProxyClass()` 方法获取代理类的 `Class` 对象。该方法接收类加载器（`ClassLoader`）和要实现的接口数组作为参数，并返回代理类的 `Class` 对象。
2.  根据获取的代理类的 `Class` 对象，使用 `Class.newInstance()` 或者 `Constructor.newInstance()` 方法创建代理类的实例。这个实例就是最终生成的代理对象。
3.  生成的代理对象会继承自 `Proxy` 类并实现目标接口，从而具备目标接口的行为。
    需要注意的是，代理类的生成过程是在运行时动态完成的。具体的实现方式可能会有所不同，可以采用字节码生成技术（如动态生成字节码），或者通过库和框架提供的工具类来实现。

在动态代理中，当调用代理对象的某个方法时，实际上会委托给 `InvocationHandler` 的 `invoke()` 方法来处理。在 `invoke()` 方法中，我们可以对方法调用进行自定义处理逻辑。
在代理对象的生成过程中，会创建一个新的代理类，该代理类继承自 `Proxy` 类，并实现了目标接口。这个代理类中会重写目标接口中的方法，以实现自定义的行为。
具体来说，当你通过动态代理调用 `cat.eat()` 方法时，会触发代理类中对应的 `eat()` 方法。这个方法会在内部调用 `InvocationHandler` 的 `invoke()` 方法，并传递相应的参数。

```java
public interface Animal {  
    void eat();  
}

public class Cat implements Animal {  
    @Override  
    public void eat() {  
        System.out.println("鱼鱼，香香");  
    }  
}

public class AnimalHandler implements InvocationHandler {  
    private Object bean;  
    public AnimalHandler(Object object){  
        this.bean = object;  
    }  
    @Override  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        System.out.println("Before invoke "  + method.getName());  
        method.invoke(bean, args);  
        System.out.println("After invoke " + method.getName());  
        return null;    
        }  
}
```

Demo:

```java
public class DynamicProxyDemo {  
    public static void main(String[] args) {  
        AnimalHandler animalHandler = new AnimalHandler(new Cat());  
        Animal cat = (Animal) Proxy.newProxyInstance(Animal.class.getClassLoader(),new Class[]{Animal.class},animalHandler);  
        cat.eat();  
    }  
}
```

在 `Proxy.newProxyInstance()` 方法中，有三个参数，分别是：

1.  类加载器（ClassLoader）：指定用于加载代理类的类加载器。代理类是在运行时动态生成的，所以需要指定一个类加载器来加载这个代理类。一般情况下，可以使用目标类的类加载器作为参数，例如 `target.getClass().getClassLoader()`。
2.  接口数组：指定代理类要实现的接口。代理对象会实现这些接口，并提供接口中定义的方法。可以传递多个接口，通过数组的形式指定，例如 `new Class<?>[] { SomeInterface.class, AnotherInterface.class }`。
3.  调用处理器（InvocationHandler）：指定代理对象的调用处理器。调用处理器是一个实现了 `InvocationHandler` 接口的对象，它负责处理代理对象的方法调用。在代理对象的方法被调用时，会委托给调用处理器的 `invoke()` 方法进行处理。通过自定义的调用处理器，可以实现自定义的逻辑，例如执行一些前置或后置操作。一般情况下，可以创建一个实现了 `InvocationHandler` 接口的类的实例，并将其作为参数传递给 `Proxy.newProxyInstance()` 方法。

`invoke(Object proxy, Method method, Object[] args)` 是 `InvocationHandler` 接口中的方法，用于处理代理对象的方法调用。

-   `proxy` 参数是代理对象本身，也就是通过动态代理生成的对象。在 `invoke()` 方法中，我们可以使用 `proxy` 对象来调用代理对象的其他方法，或者将其作为参数传递给其他方法。
-   `method` 参数是被调用的方法对象，它包含了被调用方法的信息，例如方法名、参数类型等。我们可以通过 `method` 对象获取到这些信息，并在 `invoke()` 方法中根据需要处理方法调用。
-   `args` 参数是方法调用时传递的参数数组。如果被调用方法有参数，那么这个参数数组中包含了实际传递给方法的参数值。我们可以通过 `args` 数组获取到这些参数值，并在 `invoke()` 方法中根据需要处理这些参数。
-   

# 注解的底层实现

```java
@Retention(RetentionPolicy.RUNTIME)  
@Target(ElementType.TYPE)  
public @interface MyAnnotation {  //自定义注解
        String value() default "I'm an annotation.";  
}

//Demo
@MyAnnotation  
public class AnnotationDemo {  
    public static void main(String[] args) {  
        MyAnnotation myAnnotation = AnnotationDemo.class.getDeclaredAnnotation(MyAnnotation.class);  
        System.out.println(myAnnotation.value());  
    }  
}
```

在idea中查看类继承关系图
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685627663133.png)
发现`MyAnnotation`继承自`Annotation`接口，转到接口源码。
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685627730431.png)

所以注解是什么呢？接口？类？抽象类？
看一下字节码:![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685627740206.png)
发现调用的是INVOKEINTERFACE指令，在jvm中方法调用的指令有如下四种：

1.  `invokestatic`：用于调用静态方法。这个指令会根据方法的符号引用定位到目标方法，并执行方法调用。
2.  `invokespecial`：用于调用实例构造方法（`<init>`）、私有方法以及父类的方法（包括构造方法）。这个指令同样会根据方法的符号引用定位到目标方法，并执行方法调用。
3.  `invokevirtual`：用于调用普通实例方法。这个指令会在运行时根据对象的实际类型找到对应的方法，并执行方法调用。如果目标方法是动态绑定的（即被重写的方法），会根据对象的实际类型来确定调用哪个版本的方法。
4.  `invokeinterface`：用于调用接口方法。与 `invokevirtual` 类似，不同之处在于这个指令是为了调用接口中的方法。它会在运行时根据对象的实际类型找到对应的实现类，并执行方法调用。

所以，编译器认为value()方法是一个接口方法。
所以，我们可以得出结论：注解是一个继承自Annotation接口的接口。里面的每一个属性，其实就是接口的一个抽象方法。

那么新的问题来了，如果注解是接口，那么其何时实例化，怎么实例化？  
我们是通过 AnnotationDemo.class.getDeclaredAnnotation(MyAnnotation.class);  来获取到注解的实例的，那么使用debug模式看一下这个方法。
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685627755437.png)
发现返回的实例名称 是$Proxy1, 很明显是一个代理对象，里面还有一个叫AnnotationInvocationHandler的类。
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685627764793.png)
上图就是注解的代理逻辑封装。

总结：注解@interface 是一个实现了Annotation接口的接口， 然后在调用getDeclaredAnnotations()方法的时候，返回一个代理$Proxy对象，这个是使用jdk动态代理创建，使用Proxy的newProxyInstance方法，传入接口 和InvocationHandler的一个实例(也就是 AnotationInvocationHandler ) ，最后返回一个实例。

那么Proxy的newProxyInstance方法在何处调用呢？我们继续步入。
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685627785449.png)
sun.reflect.annotation.AnnotationParser#annotationForMap
在这里jdk动态代理的newProxyInstance返回代理对象

现在，还有一个问题：一开始传给注解的参数，存储到了哪？
我们查看getDeclaredAnnotation这个方法的源码。![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685627801087.png)

进入 declaredAnnotations
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685627849575.png)
可以看到，这是Class.java里的一个静态内部类，declaredAnnotations是一个map，
我们从这个map中取出注解的代理对象。

cd ..
进入 annotationData()
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685627857362.png)
发现annotationData()返回了一个newAnnotationData。这个newAnnotationData是`AnnotationData newAnnotationData = createAnnotationData(classRedefinedCount);`创造的，（while部分是缓存，只会解析一次注解)

进入 createAnnotationData(classRedefinedCount)
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685627864206.png)
可以看到，这个方法return了一个`new AnnotationData(annotations, declaredAnnotations, classRedefinedCount)`，调用`AnnotationData`的构造函数，那么AnnotationData里的map从哪里来呢？
就在这个函数的第一行。
可以看到，`AnnotationParser.parseAnnotations(getRawAnnotations(), getConstantPool(), this);`产生了一个所需map。
需要注意的是其中的两个参数`getRawAnnotations()`和`getConstantPool()`

- getRawAnnotations():
    ![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628419881.png)
    native方法，获取原始批注。
- getConstantPool():
    ![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628437075.png)
    获取常量池 也是native方法

进入parseAnnotations方法
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628576651.png)
调用parseAnnotations2方法。
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628615509.png)

进入parseAnnotations2方法。
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628513044.png)
可以看到map在这时被填充
klass是键，由`a.annotationType()`返回,annotationType()是Annotation类的一个方法，返回该注解的class对象。
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628635090.png)
值是a本身，一个Annotation对象。
这个对象从
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628651055.png)
这个函数获得。这是一个重载的函数，我们进入这个函数。

![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628686997-20240213204304931.png)

进入parseAnnotation2另一个被重载的函数。
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628770586.png)
函数很长，函数最后有一个
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628787224.png)
是不是很眼熟，进入发现
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628801431-20240213204400469.png)
正是上面调用newProxyInstance的函数。
再转回来看annotationForMap的入参，是注解的字节码文件和一个map，这个map在上面被填入了这个注解的全部信息。
怎么填入的呢？
![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1685628808933.png)
value从`parseMemberValue(memberType, buf, constPool, container)`取得。
所以最终发现，注解的信息都放在了constpool中，在创建实例的时候，会通过getConstantPool()获取出来，是一个byte[]流，需要进行转换。

通过入参的数据，annotationForMap创造了一个代理对象，并且逐级返回，被塞进了declaredAnnotations这个map中，这个map的key是注解的class对象，value是代理对象。然后通过`annotationData().declaredAnnotations.get(annotationClass)`返回给了 ` AnnotationDemo.class.getDeclaredAnnotation(MyAnnotation.class)`，即我们在demo中调用的地方，并把返回的代理对象交给了myAnnotation。

让我们做一个总结。
注解本质是一个继承了Annotation的特殊接口，其具体实现类是Java运行时生成的动态代理类。在调用getDeclaredAnnotations()方法的时候，返回一个代理$Proxy对象，这个对象使用jdk动态代理创建，使用Proxy的newProxyInstance方法时候，传入Annotation的class对象和InvocationHandler的一个实例(也就是AnotationInvocationHandler ) ，最后返回一个代理实例。期间，在创建代理对象之前，解析注解时候 从该注解类的常量池中取出注解的信息，包括之前写到注解中的参数，然后将这些信息在创建 AnnotationInvocationHandler时候 ，传入进去 作为构造函数的参数。
通过代理对象调用自定义注解（接口）的方法，会最终调用AnnotationInvocationHandler的invoke方法。该方法会从memberValues这个Map中索引出对应的值。而memberValues的来源是Java常量池。


文章参考：
注解基础部分：

https://pdai.tech/md/java/basic/java-basic-x-annotation.html

注解底层实现部分：

https://blog.csdn.net/qq_20009015/article/details/106038023

https://juejin.cn/post/6960685149503619109

https://www.cnblogs.com/strongmore/p/13282691.html