Java注解处理器使用详解
---
在这篇文章中，我将阐述怎样写一个注解处理器(Annotation Processor)。在这篇教程中，首先，我将向您解释什么是注解器，你可以利用这个强大的工具做什么以及不能做什么；然后，我将一步一步实现一个简单的注解器。

### 一些基本概念
在开始之前，我们首先申明一个非常重要的问题：我们并不讨论那些在运行时（Runtime）通过反射机制运行处理的注解，而是讨论在编译时（Compile time）处理的注解。

注解处理器是一个在javac中的，用来编译时扫描和处理的注解的工具。你可以为特定的注解，注册你自己的注解处理器。到这里，我假设你已经知道什么是注解，并且知道怎么申明的一个注解类型。如果你不熟悉注解，你可以在这[官方文档](http://docs.oracle.com/javase/tutorial/java/annotations/index.html)中得到更多信息。注解处理器在Java 5开始就有了，但是从Java 6（2006年12月发布）开始才有可用的API。过了一些时间，Java世界才意识到注解处理器的强大作用，所以它到最近几年才流行起来。

一个注解的注解处理器，以<font color=red>Java代码</font>（或者<font color=red>编译过的字节码</font>）作为<font color=red>输入</font>，生成文件（通常是<font color=red>.java文件</font>）作为<font color=red>输出</font>。这具体的含义是什么呢？你可以生成Java代码！这些生成的Java代码是在生成的.java文件中，所以<font color=red>你不能修改已经存在的Java类</font>，例如向已有的类中添加方法。这些生成的Java文件，会同其他普通的手动编写的Java源代码一样被javac编译。

### AbstractProcessor
我们首先看一下处理器的API。每一个处理器都是继承于AbstractProcessor，如下所示：
```java
package com.example;

public class MyProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment env){ }

    @Override
    public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }

    @Override
    public Set<String> getSupportedAnnotationTypes() { }

    @Override
    public SourceVersion getSupportedSourceVersion() { }

}
```
* init(ProcessingEnvironment env): 每一个注解处理器类都必须有一个空的构造函数。然而，这里有一个特殊的init()方法，它会被注解处理工具调用，并输入ProcessingEnviroment参数。ProcessingEnviroment提供很多有用的工具类Elements, Types和Filer。后面我们将看到详细的内容。
* process(Set<? extends TypeElement> annotations, RoundEnvironment env): 这相当于每个处理器的主函数main()。你在这里写你的扫描、评估和处理注解的代码，以及生成Java文件。输入参数RoundEnviroment，可以让你查询出包含特定注解的被注解元素。后面我们将看到详细的内容。
* getSupportedAnnotationTypes(): 这里你必须指定，这个注解处理器是注册给哪个注解的。注意，它的返回值是一个字符串的集合，包含本处理器想要处理的注解类型的合法全称。换句话说，你在这里定义你的注解处理器注册到哪些注解上。
* getSupportedSourceVersion(): 用来指定你使用的Java版本。通常这里返回SourceVersion.latestSupported()。然而，如果你有足够的理由只支持Java 6的话，你也可以返回SourceVersion.RELEASE_6。我推荐你使用前者。

在Java 7中，你也可以使用注解来代替getSupportedAnnotationTypes()和getSupportedSourceVersion()，像这样：
```java
@SupportedSourceVersion(SourceVersion.latestSupported())
@SupportedAnnotationTypes({
   // 合法注解全名的集合
 })
public class MyProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment env){ }

    @Override
    public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }
}
```

因为兼容的原因，特别是针对Android平台，我建议使用重载getSupportedAnnotationTypes()和getSupportedSourceVersion()方法代替@SupportedAnnotationTypes和@SupportedSourceVersion。

接下来的你必须知道的事情是，注解处理器是运行在它自己的虚拟机JVM中的。是的，你没有看错，javac启动一个完整Java虚拟机来运行注解处理器。这对你意味着什么？你可以使用任何你在其他java应用中使用的的东西。使用guava。如果你愿意，你可以使用依赖注入工具，例如dagger或者其他你想要的类库。但是不要忘记，即使是一个很小的处理，你也要像其他Java应用一样，注意算法效率，以及[设计模式](http://www.codeceo.com/article/category/develop/design-patterns)。

### 注册你的处理器
你可能会问，我怎样处理器MyProcessor到javac中。你必须提供一个.jar文件。就像其他.jar文件一样，你打包你的注解处理器到此文件中。并且，在你的jar中，你需要打包一个特定的文件javax.annotation.processing.Processor到META-INF/services路径下。所以，你的.jar文件看起来就像下面这样：
```java
MyProcessor.jar
	- com
		- example
			- MyProcessor.class

	- META-INF
		- services
			- javax.annotation.processing.Processor
```
打包进MyProcessor.jar中的javax.annotation.processing.Processor的内容是，注解处理器的合法的全名列表，每一个元素换行分割：
```java
com.example.MyProcessor  
com.foo.OtherProcessor  
net.blabla.SpecialProcessor
```
把MyProcessor.jar放到你的builpath中，javac会自动检查和读取javax.annotation.processing.Processor中的内容，并且注册MyProcessor作为注解处理器。

### 例子：工厂模式
是时候来说一个实际的例子了。我们将使用maven工具来作为我们的编译系统和依赖管理工具。如果你不熟悉maven，不用担心，因为maven不是必须的。本例子的完成代码在[Github](https://github.com/sockeqwe/annotationprocessing101)上。

开始之前，我必须说，要为这个教程找到一个需要用注解处理器解决的简单问题，实在并不容易。这里我们将实现一个非常简单的工厂模式（不是抽象工厂模式）。这将对注解处理器的API做一个非常简明的介绍。所以，这个问题的程序并不是那么有用，也不是一个真实世界的例子。所以在此申明，你将学习关于注解处理过程的相关内容，而不是设计模式。

我们将要解决的问题是：我们将实现一个披萨店，这个披萨店给消费者提供两种披萨（“Margherita”和“Calzone”）以及提拉米苏甜点(Tiramisu)。

看一下如下的代码，不需要做任何更多的解释：
```java
public interface Meal {  
  public float getPrice();
}

public class MargheritaPizza implements Meal {

  @Override public float getPrice() {
    return 6.0f;
  }
}

public class CalzonePizza implements Meal {

  @Override public float getPrice() {
    return 8.5f;
  }
}

public class Tiramisu implements Meal {

  @Override public float getPrice() {
    return 4.5f;
  }
}
```
为了在我们的披萨店PizzsStore下订单，消费者需要输入餐(Meal)的名字。
```java
public class PizzaStore {

  public Meal order(String mealName) {

    if (mealName == null) {
      throw new IllegalArgumentException("Name of the meal is null!");
    }

    if ("Margherita".equals(mealName)) {
      return new MargheritaPizza();
    }

    if ("Calzone".equals(mealName)) {
      return new CalzonePizza();
    }

    if ("Tiramisu".equals(mealName)) {
      return new Tiramisu();
    }

    throw new IllegalArgumentException("Unknown meal '" + mealName + "'");
  }

  public static void main(String[] args) throws IOException {
    PizzaStore pizzaStore = new PizzaStore();
    Meal meal = pizzaStore.order(readConsole());
    System.out.println("Bill: $" + meal.getPrice());
  }
}
```
正如你所见，在order()方法中，我们有很多的if语句，并且如果我们每添加一种新的披萨，我们都要添加一条新的if语句。但是等一下，使用注解处理和工厂模式，我们可以让注解处理器来帮我们自动生成这些if语句。如此以来，我们期望的是如下的代码：
```java
public class PizzaStore {

  private MealFactory factory = new MealFactory();

  public Meal order(String mealName) {
    return factory.create(mealName);
  }

  public static void main(String[] args) throws IOException {
    PizzaStore pizzaStore = new PizzaStore();
    Meal meal = pizzaStore.order(readConsole());
    System.out.println("Bill: $" + meal.getPrice());
  }
}
```
MealFactory应该是如下的样子：
```java
public class MealFactory {

  public Meal create(String id) {
    if (id == null) {
      throw new IllegalArgumentException("id is null!");
    }
    if ("Calzone".equals(id)) {
      return new CalzonePizza();
    }

    if ("Tiramisu".equals(id)) {
      return new Tiramisu();
    }

    if ("Margherita".equals(id)) {
      return new MargheritaPizza();
    }

    throw new IllegalArgumentException("Unknown id = " + id);
  }
}
```

### @Factory注解
你能猜到么：我们想用注解处理器自动生成MealFactory。更一般的说，我们将想要提供一个注解和一个处理器来生成工厂类。

我们先来看一下@Factory注解：
```java
@Target(ElementType.TYPE) @Retention(RetentionPolicy.CLASS)
public @interface Factory {

  /**
   * 工厂的名字
   */
  Class type();

  /**
   * 用来表示生成哪个对象的唯一id
   */
  String id();
}
```
想法是这样的：我们将使用同样的type()注解那些属于同一个工厂的类，并且用注解的id()做一个映射，例如从"Calzone"映射到"ClzonePizza"类。我们应用@Factory注解到我们的类中，如下：
```java
@Factory(
    id = "Margherita",
    type = Meal.class
)
public class MargheritaPizza implements Meal {

  @Override public float getPrice() {
    return 6f;
  }
}
```
```java
@Factory(
    id = "Calzone",
    type = Meal.class
)
public class CalzonePizza implements Meal {

  @Override public float getPrice() {
    return 8.5f;
  }
}
```
```java
@Factory(
    id = "Tiramisu",
    type = Meal.class
)
public class Tiramisu implements Meal {

  @Override public float getPrice() {
    return 4.5f;
  }
}
```

你可能会问你自己，我们是否可以只把@Factory注解应用到我们的Meal接口上？答案是，注解是不能继承的。一个类class X被注解，并不意味着它的子类class Y extends X会自动被注解。在我们开始写处理器的代码之前，我们先规定如下一些规则：

* 只有类可以被@Factory注解，因为接口或者抽象类并不能用new操作实例化；
* 被@Factory注解的类，必须至少提供一个公开的默认构造器（即没有参数的构造函数）。否者我们没法实例化一个对象。
* 被@Factory注解的类必须直接或者间接的继承于type()指定的类型；
* 具有相同的type的注解类，将被聚合在一起生成一个工厂类。这个生成的类使用Factory后缀，例如type = Meal.class，将生成MealFactory工厂类；
* id只能是String类型，并且在同一个type组中必须唯一。

### 处理器
我将通过添加代码和一段解释的方法，一步一步的引导你来构建我们的处理器。省略号(...)表示省略那些已经讨论过的或者将在后面的步骤中讨论的代码，目的是为了让我们的代码有更好的可读性。正如我们前面说的，我们完整的代码可以在[Github](https://github.com/sockeqwe/annotationprocessing101)上找到。好了，让我们来看一下我们的处理器类FactoryProcessor的骨架：
```java
@AutoService(Processor.class)
public class FactoryProcessor extends AbstractProcessor {

  private Types typeUtils;
  private Elements elementUtils;
  private Filer filer;
  private Messager messager;
  private Map<String, FactoryGroupedClasses> factoryClasses = new LinkedHashMap<String, FactoryGroupedClasses>();

  @Override
  public synchronized void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    typeUtils = processingEnv.getTypeUtils();
    elementUtils = processingEnv.getElementUtils();
    filer = processingEnv.getFiler();
    messager = processingEnv.getMessager();
  }

  @Override
  public Set<String> getSupportedAnnotationTypes() {
    Set<String> annotataions = new LinkedHashSet<String>();
    annotataions.add(Factory.class.getCanonicalName());
    return annotataions;
  }

  @Override
  public SourceVersion getSupportedSourceVersion() {
    return SourceVersion.latestSupported();
  }

  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    ...
  }
}
```
你看到在代码的第一行是@AutoService(Processor.class)，这是什么？这是一个其他注解处理器中引入的注解。AutoService注解处理器是Google开发的，用来生成
META-INF/services/javax.annotation.processing.Processor文件的。是的，你没有看错，我们可以在注解处理器中使用注解。非常方便，难道不是么？在getSupportedAnnotationTypes()中，我们指定本处理器将处理@Factory注解。

### Elements和TypeMirrors
在init()中我们获得如下引用：
* Elements：一个用来处理Element的工具类（后面将做详细说明）；
* Types：一个用来处理TypeMirror的工具类（后面将做详细说明）；
* Filer：正如这个名字所示，使用Filer你可以创建文件。

在注解处理过程中，我们扫描所有的Java源文件。源代码的每一个部分都是一个特定类型的Element。换句话说：Element代表程序的元素，例如包、类或者方法。每个Element代表一个静态的、语言级别的构件。在下面的例子中，我们通过注释来说明这个
```java
package com.example;    // PackageElement

public class Foo {        // TypeElement

    private int a;      // VariableElement
    private Foo other;  // VariableElement

    public Foo () {}    // ExecuteableElement

    public void setA (  // ExecuteableElement
                     int newA   // TypeElement
                     ) {}
}
```
你必须换个角度来看源代码，它只是结构化的文本，他不是可运行的。你可以想象它就像你将要去解析的XML文件一样（或者是编译器中抽象的语法树）。就像XML解释器一样，有一些类似DOM的元素。你可以从一个元素导航到它的父或者子元素上。

举例来说，假如你有一个代表public class Foo类的TypeElement元素，你可以遍历它的孩子，如下：
```java
TypeElement fooClass = ... ;  
for (Element e : fooClass.getEnclosedElements()){ // iterate over children  
    Element parent = e.getEnclosingElement();  // parent == fooClass
}
```
正如你所见，Element代表的是源代码。TypeElement代表的是源代码中的类型元素，例如类。然而，TypeElement并不包含类本身的信息。你可以从TypeElement中获取类的名字，但是你获取不到类的信息，例如它的父类。这种信息需要通过TypeMirror获取。你可以通过调用elements.asType()获取元素的TypeMirror。

### 搜索@Factory注解
我们来一步一步实现process()方法。首先，我们从搜索被注解了@Factory的类开始：
```java
@AutoService(Processor.class)
public class FactoryProcessor extends AbstractProcessor {

  private Types typeUtils;
  private Elements elementUtils;
  private Filer filer;
  private Messager messager;
  private Map<String, FactoryGroupedClasses> factoryClasses = new LinkedHashMap<String, FactoryGroupedClasses>();
    ...

  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    // 遍历所有被注解了@Factory的元素
    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {
          ...
    }
  }
 ...
}
```
这里并没有什么高深的技术。roundEnv.getElementsAnnotatedWith(Factory.class))返回所有被注解了@Factory的元素的列表。你可能已经注意到，我们并没有说“所有被注解了@Factory的类的列表”，因为它真的是返回Element的列表。请记住：Element可以是类、方法、变量等。所以，接下来，我们必须检查这些Element是否是一个类：
```java
@Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {

      // 检查被注解为@Factory的元素是否是一个类
      if (annotatedElement.getKind() != ElementKind.CLASS) {
        // ...
      }
   }
   ...
}
```
为什么要这么做？我们要确保只有class元素被我们的处理器处理。前面我们已经学习到类是用TypeElement表示。我们为什么不这样判断呢
```java
// 检查被注解为@Factory的元素是否是一个类
if (! (annotatedElement instanceof TypeElement) ){
  // ...
}
```
这是错误的，因为接口（interface）类型也是TypeElement。所以在注解处理器中，我们要避免使用instanceof，而是配合TypeMirror使用EmentKind或者TypeKind。

### 错误处理
在init()中，我们也获得了一个Messager对象的引用。<font color=red>Messager提供给注解处理器一个报告错误、警告以及提示信息的途径</font>。它不是注解处理器开发者的日志工具，而是用来写一些信息给使用此注解器的第三方开发者的。在[官方文档](http://docs.oracle.com/javase/7/docs/api/javax/tools/Diagnostic.Kind.html)中描述了消息的不同级别。非常重要的是Kind.ERROR，因为这种类型的信息用来表示我们的注解处理器处理失败了。很有可能是第三方开发者错误的使用了@Factory注解（例如，给接口使用了@Factory注解）。这个概念和传统的Java应用有点不一样，在传统Java应用中我们可能就抛出一个异常Exception。如果你在process()中抛出一个异常，那么运行注解处理器的JVM将会崩溃（就像其他Java应用一样），使用我们注解处理器FactoryProcessor的第三方开发者将会从javac中得到非常难懂的出错信息，因为它包含FactoryProcessor的堆栈跟踪（Stacktace）信息。因此，注解处理器就有一个Messager类，它能够打印非常优美的错误信息。除此之外，你还可以连接到出错的元素。在像IntelliJ这种现代的IDE（集成开发环境）中，第三方开发者可以直接点击错误信息，IDE将会直接跳转到第三方开发者项目中出错的源文件的相应的行。

我们重新回到process()方法的实现。如果遇到一个非类类型被注解@Factory，我们发出一个出错信息：
```java
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {

      // 检查被注解为@Factory的元素是否是一个类
      if (annotatedElement.getKind() != ElementKind.CLASS) {
        error(annotatedElement, "Only classes can be annotated with @%s",
            Factory.class.getSimpleName());
        return true; // 退出处理
      }
      ...
    }

private void error(Element e, String msg, Object... args) {  
    messager.printMessage(
        Diagnostic.Kind.ERROR,
        String.format(msg, args),
        e);
  }

}
```
让Messager显示相关出错信息，更重要的是注解处理器程序必须完成运行而不崩溃，这就是为什么在调用error()后直接return。如果我们不直接返回，process()将继续运行，因为messager.printMessage( Diagnostic.Kind.ERROR)不会停止此进程。因此，如果我们在打印错误信息以后不返回的话，我们很可能就会运行到一个NullPointerException等。就像我们前面说的，如果我们继续运行process()，问题是如果在process()中抛出一个未处理的异常，javac将会打印出内部的NullPointerException，而不是Messager中的错误信息。

### 数据模型
在继续检查被注解@Fractory的类是否满足我们上面说的5条规则之前，我们将介绍一个让我们更方便继续处理的数据结构。有时候，一个问题或者解释器看起来如此简单，以至于程序员倾向于用一个面向过程方式来写整个处理器。但是你知道吗？一个注解处理器任然是一个Java程序，所以我们需要使用面向对象编程、接口、设计模式，以及任何你在其他普通Java程序中使用的技巧。

我们的FactoryProcessor非常简单，但是我们仍然想要把一些信息存为对象。在FactoryAnnotatedClass中，我们保存被注解类的数据，比如合法的类的名字，以及@Factory注解本身的一些信息。所以，我们保存TypeElement和处理过的@Factory注解：
```java
public class FactoryAnnotatedClass {

  private TypeElement annotatedClassElement;
  private String qualifiedSuperClassName;
  private String simpleTypeName;
  private String id;

  public FactoryAnnotatedClass(TypeElement classElement) throws IllegalArgumentException {
    this.annotatedClassElement = classElement;
    Factory annotation = classElement.getAnnotation(Factory.class);
    id = annotation.id();

    if (StringUtils.isEmpty(id)) {
      throw new IllegalArgumentException(
          String.format("id() in @%s for class %s is null or empty! that's not allowed",
              Factory.class.getSimpleName(), classElement.getQualifiedName().toString()));
    }

    // Get the full QualifiedTypeName
    try {
      Class<?> clazz = annotation.type();
      qualifiedSuperClassName = clazz.getCanonicalName();
      simpleTypeName = clazz.getSimpleName();
    } catch (MirroredTypeException mte) {
      DeclaredType classTypeMirror = (DeclaredType) mte.getTypeMirror();
      TypeElement classTypeElement = (TypeElement) classTypeMirror.asElement();
      qualifiedSuperClassName = classTypeElement.getQualifiedName().toString();
      simpleTypeName = classTypeElement.getSimpleName().toString();
    }
  }

  /**
   * 获取在{@link Factory#id()}中指定的id
   * return the id
   */
  public String getId() {
    return id;
  }

  /**
   * 获取在{@link Factory#type()}指定的类型合法全名
   *
   * @return qualified name
   */
  public String getQualifiedFactoryGroupName() {
    return qualifiedSuperClassName;
  }

  /**
   * 获取在{@link Factory#type()}{@link Factory#type()}指定的类型的简单名字
   *
   * @return qualified name
   */
  public String getSimpleFactoryGroupName() {
    return simpleTypeName;
  }

  /**
   * 获取被@Factory注解的原始元素
   */
  public TypeElement getTypeElement() {
    return annotatedClassElement;
  }
}
```
代码很多，但是最重要的部分是在构造函数中。其中你能找到如下的代码：
```java
Factory annotation = classElement.getAnnotation(Factory.class);  
id = annotation.id(); // Read the id value (like "Calzone" or "Tiramisu")

if (StringUtils.isEmpty(id)) {  
    throw new IllegalArgumentException(
          String.format("id() in @%s for class %s is null or empty! that's not allowed",
              Factory.class.getSimpleName(), classElement.getQualifiedName().toString()));
}
```
这里我们获取@Factory注解，并且检查id是否为空？如果为空，我们将抛出IllegalArgumentException异常。你可能感到疑惑的是，前面我们说了不要抛出异常，而是使用Messager。这里仍然不矛盾。我们抛出内部的异常，你在将在后面看到会在process()中捕获这个异常。我这样做出于一下两个原因：
* 我想示意我们应该像普通的Java程序一样编码。抛出和捕获异常是非常好的Java编程实践；
* 如果我们想要在FactoryAnnotatedClass中打印信息，我需要也传入Messager对象，并且我们在错误处理一节中已经提到，为了打印Messager信息，我们必须成功停止处理器运行。如果我们使用Messager打印了错误信息，我们怎样告知process()出现了错误呢？最容易，并且我认为最直观的方式就是抛出一个异常，然后让process()捕获之。
接下来，我们将获取@Fractory注解中的type成员。我们比较关心的是合法的全名：
```java
try {  
      Class<?> clazz = annotation.type();
      qualifiedGroupClassName = clazz.getCanonicalName();
      simpleFactoryGroupName = clazz.getSimpleName();
} catch (MirroredTypeException mte) {
      DeclaredType classTypeMirror = (DeclaredType) mte.getTypeMirror();
      TypeElement classTypeElement = (TypeElement) classTypeMirror.asElement();
      qualifiedGroupClassName = classTypeElement.getQualifiedName().toString();
      simpleFactoryGroupName = classTypeElement.getSimpleName().toString();
}
```
这里有一点小麻烦，因为这里的类型是一个java.lang.Class。这就意味着，他是一个真正的Class对象。因为<font color=red>注解处理是在编译Java源代码之前</font>。我们需要考虑如下两种情况：
* 这个类<font color=red>已经被编译</font>：这种情况是：如果第三方.jar包含已编译的被@Factory注解.class文件。在这种情况下，我们可以像try中那块代码中所示直接获取Class。
* 这个还<font color=red>没有被编译</font>：这种情况是我们尝试编译被@Fractory注解的源代码。这种情况下，直接获取Class会抛出MirroredTypeException异常。幸运的是，MirroredTypeException包含一个TypeMirror，它表示我们未编译类。因为我们已经知道它必定是一个类类型（我们已经在前面检查过），我们可以直接强制转换为DeclaredType，然后读取TypeElement来获取合法的名字。
好了，我们现在还需要一个数据结构FactoryGroupedClasses，它将简单的组合所有的FactoryAnnotatedClasses到一起。
```java
public class FactoryGroupedClasses {

  private String qualifiedClassName;

  private Map<String, FactoryAnnotatedClass> itemsMap =
      new LinkedHashMap<String, FactoryAnnotatedClass>();

  public FactoryGroupedClasses(String qualifiedClassName) {
    this.qualifiedClassName = qualifiedClassName;
  }

  public void add(FactoryAnnotatedClass toInsert) throws IdAlreadyUsedException {

    FactoryAnnotatedClass existing = itemsMap.get(toInsert.getId());
    if (existing != null) {
      throw new IdAlreadyUsedException(existing);
    }

    itemsMap.put(toInsert.getId(), toInsert);
  }

  public void generateCode(Elements elementUtils, Filer filer) throws IOException {
    ...
  }
}
```
正如你所见，这是一个基本的Map<String, FactoryAnnotatedClass>，这个映射表用来映射@Factory.id()到FactoryAnnotatedClass。我们选择Map这个数据类型，是因为我们要确保每个id是唯一的，我们可以很容易通过map查找实现。generateCode()方法将被用来生成工厂类代码（将在后面讨论）。

### 匹配标准
我们继续实现process()方法。接下来我们想要检查被注解的类必须有只要一个公开的构造函数，不是抽象类，继承于特定的类型，以及是一个公开类：
```java
public class FactoryProcessor extends AbstractProcessor {

  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {

      ...

      // 因为我们已经知道它是ElementKind.CLASS类型，所以可以直接强制转换
      TypeElement typeElement = (TypeElement) annotatedElement;

      try {
        FactoryAnnotatedClass annotatedClass =
            new FactoryAnnotatedClass(typeElement); // throws IllegalArgumentException

        if (!isValidClass(annotatedClass)) {
          return true; // 已经打印了错误信息，退出处理过程
         }
       } catch (IllegalArgumentException e) {
        // @Factory.id()为空
        error(typeElement, e.getMessage());
        return true;
       }
          ...
   }

 private boolean isValidClass(FactoryAnnotatedClass item) {

    // 转换为TypeElement, 含有更多特定的方法
    TypeElement classElement = item.getTypeElement();

    if (!classElement.getModifiers().contains(Modifier.PUBLIC)) {
      error(classElement, "The class %s is not public.",
          classElement.getQualifiedName().toString());
      return false;
    }

    // 检查是否是一个抽象类
    if (classElement.getModifiers().contains(Modifier.ABSTRACT)) {
      error(classElement, "The class %s is abstract. You can't annotate abstract classes with @%",
          classElement.getQualifiedName().toString(), Factory.class.getSimpleName());
      return false;
    }

    // 检查继承关系: 必须是@Factory.type()指定的类型子类
    TypeElement superClassElement =
        elementUtils.getTypeElement(item.getQualifiedFactoryGroupName());
    if (superClassElement.getKind() == ElementKind.INTERFACE) {
      // 检查接口是否实现了                                       if(!classElement.getInterfaces().contains(superClassElement.asType())) {
        error(classElement, "The class %s annotated with @%s must implement the interface %s",
            classElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
            item.getQualifiedFactoryGroupName());
        return false;
      }
    } else {
      // 检查子类
      TypeElement currentClass = classElement;
      while (true) {
        TypeMirror superClassType = currentClass.getSuperclass();

        if (superClassType.getKind() == TypeKind.NONE) {
          // 到达了基本类型(java.lang.Object), 所以退出
          error(classElement, "The class %s annotated with @%s must inherit from %s",
              classElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
              item.getQualifiedFactoryGroupName());
          return false;
        }

        if (superClassType.toString().equals(item.getQualifiedFactoryGroupName())) {
          // 找到了要求的父类
          break;
        }

        // 在继承树上继续向上搜寻
        currentClass = (TypeElement) typeUtils.asElement(superClassType);
      }
    }

    // 检查是否提供了默认公开构造函数
    for (Element enclosed : classElement.getEnclosedElements()) {
      if (enclosed.getKind() == ElementKind.CONSTRUCTOR) {
        ExecutableElement constructorElement = (ExecutableElement) enclosed;
        if (constructorElement.getParameters().size() == 0 && constructorElement.getModifiers()
            .contains(Modifier.PUBLIC)) {
          // 找到了默认构造函数
          return true;
        }
      }
    }

    // 没有找到默认构造函数
    error(classElement, "The class %s must provide an public empty default constructor",
        classElement.getQualifiedName().toString());
    return false;
  }
}
```
我们这里添加了isValidClass()方法，来检查是否我们所有的规则都被满足了：
* 必须是公开类：classElement.getModifiers().contains(Modifier.PUBLIC)
* 必须是非抽象类：classElement.getModifiers().contains(Modifier.ABSTRACT)
* 必须是@Factoy.type()指定的类型的子类或者接口的实现：首先我们使用elementUtils.getTypeElement(item.getQualifiedFactoryGroupName())创建一个传入的Class(@Factoy.type())的元素。是的，你可以仅仅通过已知的合法类名来直接创建TypeElement（使用TypeMirror）。接下来我们检查它是一个接口还是一个类：superClassElement.getKind() == ElementKind.INTERFACE。所以我们这里有两种情况：如果是接口，就判断classElement.getInterfaces().contains(superClassElement.asType())；如果是类，我们就* 必须使用currentClass.getSuperclass()扫描继承层级。注意，整个检查也可以使用typeUtils.isSubtype()来实现。
* 类必须有一个公开的默认构造函数：我们遍历所有的闭元素classElement.getEnclosedElements()，然后检查ElementKind.CONSTRUCTOR、Modifier.PUBLIC以及constructorElement.getParameters().size() == 0。
如果所有这些条件都满足，isValidClass()返回true，否者就打印错误信息，否则返回false。

### 组合被注解的类
一旦我们检查isValidClass()成功，我们将添加FactoryAnnotatedClass到对应的FactoryGroupedClasses中，如下：
```java
public class FactoryProcessor extends AbstractProcessor {

   private Map<String, FactoryGroupedClasses> factoryClasses =
      new LinkedHashMap<String, FactoryGroupedClasses>();

 @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
      ...
      try {
        FactoryAnnotatedClass annotatedClass =
            new FactoryAnnotatedClass(typeElement); // throws IllegalArgumentException

          if (!isValidClass(annotatedClass)) {
          return true; // 错误信息被打印，退出处理流程
        }

        // 所有检查都没有问题，所以可以添加了
        FactoryGroupedClasses factoryClass =
        factoryClasses.get(annotatedClass.getQualifiedFactoryGroupName());
        if (factoryClass == null) {
          String qualifiedGroupName = annotatedClass.getQualifiedFactoryGroupName();
          factoryClass = new FactoryGroupedClasses(qualifiedGroupName);
          factoryClasses.put(qualifiedGroupName, factoryClass);
        }

        // 如果和其他的@Factory标注的类的id相同冲突，
        // 抛出IdAlreadyUsedException异常
        factoryClass.add(annotatedClass);
      } catch (IllegalArgumentException e) {
        // @Factory.id()为空 --> 打印错误信息
        error(typeElement, e.getMessage());
        return true;
      } catch (IdAlreadyUsedException e) {
        FactoryAnnotatedClass existing = e.getExisting();
        // 已经存在
        error(annotatedElement,
            "Conflict: The class %s is annotated with @%s with id ='%s' but %s already uses the same id",
            typeElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
            existing.getTypeElement().getQualifiedName().toString());
        return true;
      }
    }
    ...
}
```

### 代码生成
我们已经收集了所有被@Factory注解的类保存为FactoryAnnotatedClass，并且组合到了FactoryGroupedClasses。现在我们将为每个工厂生成Java文件了：
```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {  
    ...
  try {
        for (FactoryGroupedClasses factoryClass : factoryClasses.values()) {
          factoryClass.generateCode(elementUtils, filer);
        }
    } catch (IOException e) {
        error(null, e.getMessage());
    }

    return true;
}
```
写Java文件，和写其他普通文件没有什么两样。使用Filer提供的Writer对象，我们可以连接字符串来写我们生成的Java代码。幸运的是，Square公司（因为提供了许多非常优秀的开源项目而非常有名）给我们提供了JavaWriter，这是一个高级的生成Java代码的库：
```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {  
    ...
  try {
        for (FactoryGroupedClasses factoryClass : factoryClasses.values()) {
          factoryClass.generateCode(elementUtils, filer);
        }
    } catch (IOException e) {
        error(null, e.getMessage());
    }

    return true;
}
```
写Java文件，和写其他普通文件没有什么两样。使用Filer提供的Writer对象，我们可以连接字符串来写我们生成的Java代码。幸运的是，Square公司（因为提供了许多非常优秀的开源项目二非常有名）给我们提供了JavaWriter，这是一个高级的生成Java代码的库：

```java
public class FactoryGroupedClasses {

  /**
   * 将被添加到生成的工厂类的名字中
   */
  private static final String SUFFIX = "Factory";

  private String qualifiedClassName;

  private Map<String, FactoryAnnotatedClass> itemsMap =
      new LinkedHashMap<String, FactoryAnnotatedClass>();
    ...

  public void generateCode(Elements elementUtils, Filer filer) throws IOException {

    TypeElement superClassName = elementUtils.getTypeElement(qualifiedClassName);
    String factoryClassName = superClassName.getSimpleName() + SUFFIX;

    JavaFileObject jfo = filer.createSourceFile(qualifiedClassName + SUFFIX);
    Writer writer = jfo.openWriter();
    JavaWriter jw = new JavaWriter(writer);

    // 写包名
    PackageElement pkg = elementUtils.getPackageOf(superClassName);
    if (!pkg.isUnnamed()) {
      jw.emitPackage(pkg.getQualifiedName().toString());
      jw.emitEmptyLine();
    } else {
      jw.emitPackage("");
    }

    jw.beginType(factoryClassName, "class", EnumSet.of(Modifier.PUBLIC));
    jw.emitEmptyLine();
    jw.beginMethod(qualifiedClassName, "create", EnumSet.of(Modifier.PUBLIC), "String", "id");

    jw.beginControlFlow("if (id == null)");
    jw.emitStatement("throw new IllegalArgumentException(\"id is null!\")");
    jw.endControlFlow();

    for (FactoryAnnotatedClass item : itemsMap.values()) {
      jw.beginControlFlow("if (\"%s\".equals(id))", item.getId());
      jw.emitStatement("return new %s()", item.getTypeElement().getQualifiedName().toString());
      jw.endControlFlow();
      jw.emitEmptyLine();
    }

    jw.emitStatement("throw new IllegalArgumentException(\"Unknown id = \" + id)");
    jw.endMethod();
    jw.endType();
    jw.close();
  }
}
```
* 注意：因为JavaWriter非常非常的流行，所以很多处理器、库、工具都依赖于JavaWriter。如果你使用依赖管理工具，例如maven或者gradle，假如一个库依赖的JavaWriter的版本比其他的库新，这将会导致一些问题。所以我建议你直接拷贝重新打包JavaWiter到你的注解处理器代码中（实际它只是一个Java文件）。
更新：JavaWrite现在已经被[JavaPoet](https://github.com/square/javapoet)取代了

### 处理循环
注解处理过程可能会多于一次。官方javadoc定义处理过程如下：

* 注解处理过程是一个有序的循环过程。在每次循环中，一个处理器可能被要求去处理那些在上一次循环中产生的源文件和类文件中的注解。第一次循环的输入是运行此工具的初始输入。这些初始输入，可以看成是虚拟的第0次的循环的输出。

一个简单的定义：一个处理循环是调用一个注解处理器的process()方法。对应到我们的工厂模式的例子中：FactoryProcessor被初始化一次（不是每次循环都会新建处理器对象），然而，如果生成了新的源文件process()能够被调用多次。听起来有点奇怪不是么？原因是这样的，这些生成的文件中也可能包含@Factory注解，它们还将会被FactoryProcessor处理。

例如我们的PizzaStore的例子中将会经过3次循环处理：


| Round | Input | Output |
|-------|-------|--------|
| 1     | CalzonePizza.java,Tiramisu.java,MargheritaPizza.java,Meal.java,PizzaStore.java | MealFactory.java |
| 2     | MealFactory.java | none |
| 3     | none | none |


我解释处理循环还有另外一个原因。如果你看一下我们的FactoryProcessor代码你就能注意到，我们收集数据和保存它们在一个私有的域中Map<String, FactoryGroupedClasses> factoryClasses。在第一轮中，我们检测到了MagheritaPizza, CalzonePizza和Tiramisu，然后生成了MealFactory.java。在第二轮中把MealFactory作为输入。因为在MealFactory中没有检测到@Factory注解，我们预期并没有错误，然而我们得到如下的信息：
```java
 Attempt to recreate a file for type com.hannesdorfmann.annotationprocessing101.factory.MealFactory
```
这个问题是因为我们没有清除factoryClasses，这意味着，在第二轮的process()中，任然保存着第一轮的数据，并且会尝试生成在第一轮中已经生成的文件，从而导致这个错误的出现。在我们的这个场景中，我们知道只有在第一轮中检查@Factory注解的类，所以我们可以简单的修复这个问题，如下：
```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {  
    try {
      for (FactoryGroupedClasses factoryClass : factoryClasses.values()) {
        factoryClass.generateCode(elementUtils, filer);
      }

      // 清除factoryClasses
      factoryClasses.clear();

    } catch (IOException e) {
      error(null, e.getMessage());
    }
    ...
    return true;
}
```
我知道这有其他的方法来处理这个问题，例如我们也可以设置一个布尔值标签等。关键的点是：我们要记住注解处理过程是需要经过多轮处理的，并且你不能重载或者重新创建已经生成的源代码。
### 分离处理器和注解
如果你已经看了我们的[代码库](https://github.com/sockeqwe/annotationprocessing101)，你将发现我们组织我们的代码到两个maven模块中了。我们这么做是因为，我们想让我们的工厂模式的例子的使用者，在他们的工程中只编译注解，而包含处理器模块只是为了编译。有点晕？我们举个例子，如果我们只有一个包。如果另一个开发者想要把我们的工厂模式处理器用于他的项目中，他就必须包含@Factory注解和整个FactoryProcessor的代码（包括FactoryAnnotatedClass和FactoryGroupedClasses）到他们项目中。我非常确定的是，他并不需要在他已经编译好的项目中包含处理器相关的代码。如果你是一个Android的开发者，你肯定听说过65k个方法的限制（即在一个.dex文件中，只能寻址65000个方法）。如果你在FactoryProcessor中使用guava，并且把注解和处理器打包在一个包中，这样的话，Android APK安装包中不只是包含FactoryProcessor的代码，而也包含了整个guava的代码。Guava有大约20000个方法。所以分开注解和处理器是非常有意义的。
### 生成的类的实例化
你已经看到了，在这个PizzaStore的例子中，生成了MealFactory类，它和其他手写的Java类没有任何区别。进而，你需要的就是像其他Java对象，手动实例化它：
```java
public class PizzaStore {

  private MealFactory factory = new MealFactory();

  public Meal order(String mealName) {
    return factory.create(mealName);
  }
  ...
}
```
* 注意：使用apt的方式而不是provided的方式使用注解库。具体原因[请看](http://www.jianshu.com/p/b5cc2418a712)
如果你是一个Android的开发者，你应该也非常熟悉一个叫做ButterKnife的注解处理器。在ButterKnife中，你使用@InjectView注解Android的View。ButterKnifeProcessor生成一个MyActivity$$ViewInjector，但是在ButterKnife你不需要手动调用new MyActivity$$ViewInjector()实例化一个ButterKnife注入的对象，而是使用Butterknife.inject(activity)。ButterKnife内部使用反射机制来实例化MyActivity$$ViewInjector()对象：
```java
try {  
    Class<?> injector = Class.forName(clsName + "$$ViewInjector");
} catch (ClassNotFoundException e) { ... }
```
但是反射机制不是很慢么，我们使用注解处理来生成本地代码，会不会导致很多的反射性能的问题？的确，反射机制的性能确实是一个问题。然而，它不需要手动去创建对象，确实提高了开发者的开发速度。ButterKnife中有一个哈希表HashMap来缓存实例化过的对象。所以MyActivity$$ViewInjector只是使用反射机制实例化一次，第二次需要MyActivity$$ViewInjector的时候，就直接冲哈希表中获得。

[FragmentArgs](https://github.com/sockeqwe/fragmentargs)非常类似于[ButterKnife](https://github.com/JakeWharton/butterknife)。它使用反射机制来创建对象，而不需要开发者手动来做这些。[FragmentArgs](https://github.com/sockeqwe/fragmentargs)在处理注解的时候生成一个特别的查找表类，它其实就是一种哈希表，所以整个[FragmentArgs](https://github.com/sockeqwe/fragmentargs)库只是在第一次使用的时候，执行一次反射调用，一旦整个Class.forName()的Fragemnt的参数对象被创建，后面的都是本地代码运行了。

作为一个注解注解处理器的开发者，这些都由你来决定，为其他的注解器使用者，在反射和可用性上找到一个好的平衡。

### 总结
到此，我希望你对注解处理过程有一个非常深刻的理解。我必须再次说明一下：注解处理器是一个非常强大的工具，减少了很多无聊的代码的编写。我也想提醒的是，注解处理器可以做到比我上面提到的工厂模式的例子复杂很多的事情。例如，泛型的类型擦除，因为注解处理器是发生在类型擦除（type erasure）之前的（译者注：类型擦除可以参考[这里](http://justjavac.iteye.com/blog/1741638)）。就像你所看到的，你在写注解处理的时候，有两个普遍的问题你需要处理：第一问题， 如果你想在其他类中使用ElementUtils, TypeUtils和Messager，你就必须把他们作为参数传进去。在我为Android开发的注解器[AnnotatedAdapter](https://github.com/sockeqwe/AnnotatedAdapter)中，我尝试使用Dagger（一个依赖注入库）来解决这个问题。在这个简单的处理中使用它听起来有点过头了，但是它确实很好用；第二个问题，你必须做查询Elements的操作。就像我之前提到的，处理Element就解析XML或者HTML一样。对于HTML你可以是用jQuery，如果在注解处理器中，有类似于jQuery的库那那绝对是酷毙了。如果你知道有类似的库，请在下面的评论告诉我。

请注意的是，在FactoryProcessor代码中有一些缺陷和陷阱。这些“错误”是我故意放进去的，是为了演示一些在开发过程中的常见错误（例如“Attempt to recreate a file”）。如果你想基于FactoryProcessor写你自己注解处理器，请不要直接拷贝粘贴这些陷阱过去，你应该从最开始就避免它们。

作者在youtube上关于Annotation Processing的演讲[Annotation Processing 101](https://www.youtube.com/watch?v=43FFfTyDYEg)

### 版权
* [翻译原文出处](http://www.codeceo.com/article/java-annotation-processor.html)
* [英文原文](http://hannesdorfmann.com/annotation-processing/annotationprocessing101/)
