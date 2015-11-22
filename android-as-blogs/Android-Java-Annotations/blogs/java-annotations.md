Java 注解指导手册 – 终极向导
---
* [Java 注解指导手册 – 终极向导](blogs/java-annotations.md)


* 翻译 by [Toien Liu](http://ifeve.com/java-annotations-tutorial/)
* 原文 by [Dani Buiza](http://www.javacodegeeks.com/2014/11/java-annotations-tutorial.html)

编者的话：注解是java的一个主要特性且每个java开发者都应该知道如何使用它

我们已经在Java Code Geeks提供了丰富的教程, 如[Creating Your Own Java Annotations](http://www.javacodegeeks.com/2014/07/creating-your-own-java-annotations.html), [Java Annotations Tutorial with Custom Annotation](http://www.javacodegeeks.com/2012/11/java-annotations-tutorial-with-custom-annotation.html) 和 [Java Annotations: Explored & Explained](http://www.javacodegeeks.com/2012/08/java-annotations-explored-explained.html).

我们也有些文章是关于注解在不同类库中的应用，包括 [Make your Spring Security @Secured annotations more DRY](http://www.javacodegeeks.com/2012/06/make-your-spring-security-secured.html)和 [Java Annotations & A Real World Spring Example](http://www.javacodegeeks.com/2012/01/java-annotations-real-world-spring.html).

现在，是时候汇总这些和注解相关的信息到一篇文章了，祝大家阅读愉快。
* 什么是注解
* 介绍
* 消费器
* 注解语法和注解元素
* 在什么地方使用
* 使用案例
* 内建注解
* Java 8 与注解
* 自定义注解
* 提取注解
* 注解集成
* 使用注解的知名类库
* 小结
* 下载
* 资料
在这篇文章中我们将阐述什么是Java注解，它们如何工作，怎么使用它们。

我们将揭开Java注解的面纱，包括内建注解或称元注解，还将讨论Java8中与之相关的的新特性。

最后，我们将实现自定义的注解，编写一个使用注解的处理程序（消费器），它通过java反射使用注解。

我们还会列出一些基于注解，知名且被广泛应用的第三方类库如：Junit，JAXB，Spring，Hibernate。

在文章的最后，会有一个压缩文件包含了文章中的所有示例，实现这些例子使用的软件版本如下所示：

* Eclipse Luna 4.4
* JRE Update 8.20
* Junit 4
* Hibernate 4.3.6
* FindBugs 3.0.0

### 什么是注解？
注解早在J2SE1.5就被引入到Java中，主要提供一种机制，这种机制允许程序员在编写代码的同时可以直接编写元数据。

在引入注解之前，程序员们描述其代码的形式尚未标准化，每个人的做法各异：transient关键字、注释、接口等。这显然不是一种优雅的方式，随之而来的一种崭新的记录元数据的形式——注解被引入到Java中。

其它因素也促成了这个决定：当时不同类型的应用程序使用XML作为标准的代码配置机制，这其实并不是最佳方式，因为代码和XML的解耦以及未来对这种解耦应用的维护并不低廉。另外，由于非保留字的使用，例如“@deprecated”自从Java1.4便开始在Java文档中使用。我非常确定这是一个现在在注解中使用“@”原因。

包含注解的设计和开发的Java规范主要有以下两篇：

* [JSR 175 A metadata facility for the Java programming Language](https://www.jcp.org/aboutJava/communityprocess/final/jsr175/index.html)
* [JSR 250 Common Annotations for the Java Platform](https://jcp.org/en/jsr/detail?id=250)

### 介绍
解释何为注解的最佳方式就是元数据这个词：描述数据自身的数据。注解就是代码的元数据，他们包含了代码自身的信息。

注解可以被用在包，类，方法，变量，参数上。自Java8起，有一种注解几乎可以被放在代码的任何位置，叫做类型注解。我们将会在后面谈到具体用法。

被注解的代码并不会直接被注解影响。这只会向第三系统提供关于自己的信息以用于不同的需求。

注解会被编译至class文件中，而且会在运行时被处理程序提取出来用于业务逻辑。当然，创建在运行时不可用的注解也是可能的，甚至可以创建只在源文件中可用，在编译时不可用的注解。

### 消费器
理解注解的目的以及如何使用它都会带来困难，因为注解本身并不包含任何功能逻辑，它们也不会影响自己注解的代码，那么，它们到底为什么而存在呢？

这个问题的解释就是我所称的注解消费器。它们是利用被注解代码并根据注解信息产生不同行为的系统或者应用程序。

例如，在Java自带的内建注解（元注解）中，消费器是执行被注解代码的JVM。还有其他稍后谈到的其他例子，例如JUnit，消费器是读取，分析被注解代码的JUnit处理程序，它还可以决定测试单元和方法执行顺序。我们会在JUnit章节更深入。

消费器使用Java中的反射机制来读取和分析被注解的源代码。使用的主要的包有：java.lang, java.lang.reflect。我们将会在本篇指南中介绍如何用反射从头开始创建一个自定义的消费器。

### 注解语法和元素
声明一个注解需要使用“@”作为前缀，这便向编译器说明，该元素为注解。例如：
```java
@Annotation
public void annotatedMehod() {
...
 }
```
上述的注解名称为Annotation，它正在注解annotatedMethod方法。编译器会处理它。注解可以以键值对的形式持有有很多元素，即注解的属性。
```java
@Annotation(
 info = "I am an annotation",
 counter = "55"
)
public void annotatedMehod() {
...
 }
```
如果注解只包含一个元素（或者只需要指定一个元素的值，其它则使用默认值），可以像这样声明
```java
@Annotation("I am an annotation")
public void annotatedMehod() {
...
 }
```
就像我们看到的一样，如果没有元素需要被指定，则不需要括号。多个注解可以使用在同一代码上，例如类：
```java
@ Annotation (info = "U a u O")
@ Annotation2
class AnnotatedClass { ... }
```
一些java本身提供的开箱即用的注解，我们称之为内建注解。也可以定义你自己的注解，称之为自定义注解。我们会在下一章讨论。

### 在什么地方使用
注解基本上可以在Java程序的每一个元素上使用：类，域，方法，包，变量，等等。

自Java8，诞生了通过类型注解的理念。在此之前，注解是限于在前面讨论的元素的声明上使用。从此，无论是类型还是声明都可以使用注解，就像：

```java
@MyAnnotation String str = "danibuiza";
```

我们将会在Java8关联章节看到这种机制的更多细节。
### 使用案例

注解可以满足许多要求，最普遍的是：

* 向编译器提供信息：注解可以被编译器用来根据不同的规则产生警告，甚至错误。一个例子是Java8中@FunctionalInterface注解，这个注解使得编译器校验被注解的类，检查它是否是一个正确的函数式接口。
* 文档：注解可以被软件应用程序计算代码的质量例如：FindBugs，PMD或者自动生成报告，例如：用来Jenkins, Jira，Teamcity。
* 代码生成：注解可以使用代码中展现的元数据信息来自动生成代码或者XML文件，一个不错的例子是JAXB。
* 运行时处理：在运行时检查的注解可以用做不同的目的，像单元测试（JUnit），依赖注入（Spring），校验，日志（Log4j），数据访问（Hibernate）等等。

在这篇手册中我们将展现几种注解可能的用法，包括流行的Java类库是如何使用它们的。

### 内建注解

Java语言自带了一系列的注解。在本章中我们将阐述最重要的一部分。这个清单只涉及了Java语言最核心的包，未包含标准JRE中所有包和库如JAXB或Servlet规范。

以下讨论到的注解中有一些被称之为Meta注解，它们的目的注解其他注解，并且包含关于其它注解的信息。

* @Retention：这个注解注在其他注解上，并用来说明如何存储已被标记的注解。这是一种元注解，用来标记注解并提供注解的信息。可能的值是：
  * SOURCE：表明这个注解会被编译器忽略，并只会保留在源代码中。
  * CLASS:表明这个注解会通过编译驻留在CLASS文件，但会被JVM在运行时忽略,正因为如此,其在运行时不可见。
  * RUNTIME：表示这个注解会被JVM获取，并在运行时通过反射获取。

我们会在稍后展开几个例子。


* @Target：这个注解用于限制某个元素可以被注解的类型。例如：
  * ANNOTATION_TYPE 表示该注解可以应用到其他注解上
  * CONSTRUCTOR 表示可以使用到构造器上
  * FIELD 表示可以使用到域或属性上
  * LOCAL_VARIABLE表示可以使用到局部变量上。
  * METHOD可以使用到方法级别的注解上。
  * PACKAGE可以使用到包声明上。
  * PARAMETER可以使用到方法的参数上
  * TYPE可以使用到一个类的任何元素上。

* @Documented：被注解的元素将会作为Javadoc产生的文档中的内容。注解都默认不会成为成为文档中的内容。这个注解可以对其它注解使用。
* @Inherited：在默认情况下，注解不会被子类继承。被此注解标记的注解会被所有子类继承。这个注解可以对类使用。
* @Deprecated：说明被标记的元素不应该再度使用。这个注解会让编译器产生警告消息。可以使用到方法，类和域上。相应的解释和原因，包括另一个可取代的方法应该同时和这个注解使用。
* @SuppressWarnings：说明编译器不会针对指定的一个或多个原因产生警告。例如：如果我们不想因为存在尚未使用的私有方法而得到警告可以这样做：
```java
@SuppressWarnings( "unused")
private String myNotUsedMethod(){
 ...
}
```
通常,编译器会因为没调用该方而产生警告; 用了注解抑制了这种行为。该注解需要一个或多个参数来指定抑制的警告类型。

* @Override：向编译器说明被注解元素是重写的父类的一个元素。在重写父类元素的时候此注解并非强制性的，不过可以在重写错误时帮助编译器产生错误以提醒我们。比如子类方法的参数和父类不匹配，或返回值类型不同。
* @SafeVarargs：断言方法或者构造器的代码不会对参数进行不安全的操作。在Java的后续版本中，使用这个注解时将会令编译器产生一个错误在编译期间防止潜在的不安全操作。

更多信息请参考：http://docs.oracle.com/javase/7/docs/api/java/lang/SafeVarargs.html

### Java 8 与注解

Java8带来了一些优势，同样注解框架的能力也得到了提升。在本章我们将会阐述，并就java8带来的3个注解做专题说明和举例：

* @Repeatable注解，关于类型注解的声明，函数式接口注解@FunctionalInterface（与Lambdas结合使用）。

@Repeatable：说明该注解标识的注解可以多次使用到同一个元素的声明上。
看一个使用的例子。首先我们创造一个能容纳重复的注解的容器：
```java
/**
 * Container for the {@link CanBeRepeated} Annotation containing a list of values
*/
@Retention( RetentionPolicy.RUNTIME )
@Target( ElementType.TYPE_USE )
public @interface RepeatedValues
{
 CanBeRepeated[] value();
}
```
接着，创建注解本身，然后标记@Repeatable
```java
@Retention( RetentionPolicy.RUNTIME )
@Target( ElementType.TYPE_USE )
@Repeatable( RepeatedValues.class )
public @interface CanBeRepeated
{
 String value();
}
```
最后，我们可以这样重复地使用：
```java
@CanBeRepeated( "the color is green" )
@CanBeRepeated( "the color is red" )
@CanBeRepeated( "the color is blue" )
public class RepeatableAnnotated
{

}
```
如果我们尝试去掉@Repeatable
```java
@Retention( RetentionPolicy.RUNTIME )
@Target( ElementType.TYPE_USE )
public @interface CannotBeRepeated
{

 String value();
}

@CannotBeRepeated( "info" )
/*
 * if we try repeat the annotation we will get an error: Duplicate annotation of non-repeatable type
 *
 * @CannotBeRepeated. Only annotation types marked
 *
 * @Repeatable can be used multiple times at one target.
 */
// @CannotBeRepeated( "more info" )
public class RepeatableAnnotatedWrong
{

}
```
我们会得到编译器的错误信息：
```java
Duplicate annotation of non-repeatable type
```

* 自Java8开始，我们可以在类型上使用注解。由于我们在任何地方都可以使用类型，包括 new操作符，casting，implements，throw等等。注解可以改善对Java代码的分析并且保证更加健壮的类型检查。这个例子说明了这一点：
```java
@SuppressWarnings( "unused" )
public static void main( String[] args )
{
 // type def
 @TypeAnnotated
 String cannotBeEmpty = null;

 // type
 List<@TypeAnnotated String> myList = new ArrayList<String>();

 // values
 String myString = new @TypeAnnotated String( "this is annotated in java 8" );

}
 // in method params
public void methodAnnotated( @TypeAnnotated int parameter )
{
 System.out.println( "do nothing" );
}
```
所有的这些在Java8之前都是不可能的。

* @FunctionalInterface：这个注解表示一个函数式接口元素。函数式接口是一种只有一个抽象方法（非默认）的接口。编译器会检查被注解元素，如果不符，就会产生错误。例子如下
```java
// implementing its methods
@SuppressWarnings( "unused" )
MyCustomInterface myFuncInterface = new MyCustomInterface()
{

 @Override
 public int doSomething( int param )
 {
 return param * 10;
 }
};

// using lambdas
@SuppressWarnings( "unused" )
 MyCustomInterface myFuncInterfaceLambdas = ( x ) -> ( x * 10 );
}

@FunctionalInterface
interface MyCustomInterface
{
/*
 * more abstract methods will cause the interface not to be a valid functional interface and
 * the compiler will thrown an error:Invalid '@FunctionalInterface' annotation;
 * FunctionalInterfaceAnnotation.MyCustomInterface is not a functional interface
 */
 // boolean isFunctionalInterface();

 int doSomething( int param );
}
```
这个注解可以被使用到类，接口，枚举和注解本身。它的被JVM保留并在runtime可见，这个是它的声明：
```java
@Documented
 @Retention(value=RUNTIME)
 @Target(value=TYPE)
public @interface FunctionalInterface
```
