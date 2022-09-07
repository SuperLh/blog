# Scala

## 语言分类

- 编译型语言：C

  - 程序员使用文本编辑器创建源代码文件
  - 编译器把源代码编译成中间代码（机器语言），并把结果放到目标代码文件中
  - 链接器把中间代码和系统的标准启动代码，库函数代码合并成可执行文件，并交由CPU执行

- 解释型语言：Python

  - 程序员使用文本编辑器创建源代码文件
  - 解释器从上到下逐行读取源代码，读取一行，翻译一行，并把翻译结果（机器语言）交由CPU执行

- 优缺点比较

  - 执行速度
    - 编译型语言执行的时候，CPU可直接读取可执行代码（机器语言），速度更快
    - 解释型语言执行的时候，需要解释器翻译一行，CPU执行一行，速度相对较慢
  - 跨平台
    - 编译型语言，不仅需要根据不同CPU安装对应的编译器，还需要根据不同操作系统选用不同启动代码，不够便利
    - 解释型语言，仅需要根据不同操作系统安装对应解释器即可，十分便利

- Java

  - <font color ='red'>Java属于一种混合体，他需要先将源代码进行编译（编译型），然后将编译后的字节码文件交给JVM，JVM读一行，解释一行（解释型），所以Java是一种先编译，后解释的语言</font>

- Scala

  - <font color='red'>Scala on JVM，Scala通过自己的编译器，生成字节码文件，然后交给JVM执行</font>

  

## 模型分类

- 面向过程：基本类型 + 指针
- 面向对象：基本类型 + 对象类型
- 函数式：基本类型 + 对象类型 + 函数



## 语法

- var/val

  ```scala
  // var：变量（允许修改）
  // val：常量（相当于final，不允许修改）
  object Test {
    def main(args: Array[String]): Unit = {
  
      var a = 4;
      val b = 5;
  
      a = 10;
      // 编译器提示错误：Reassignment to val
      b = 11;
    }
  }
  ```

- object/class内可以书写语句，会当成构造方法处理

  ```scala
  // object首先加载成员变量People，所以会首先输出People中的构造方法
  // 加载完成员变量之后，object进行初始化，会输出自己的构造方法
  // 根据方法的调用顺序进行相应的输出
  object Test {
    private val people = new People()
    println("---object start---")
    def main(args: Array[String]): Unit = {
      println("---hello world---")
      people.printMsg("zhangsan")
    }
    println("---object end---")
  }
  
  class People {
    println("---class start---")
    def printMsg(name : String): Unit = {
      println(s"---$name---")
    }
    println("---class end---")
  }
  
  // 以上代码输出结果
  // ---class start---
  // ---class end---
  // ---object start---
  // ---object end---
  // ---hello world---
  // ---zhangsan---
  ```

- string变量拼接

  ```scala
  // scala可以用s"xxx$变量名"的方式进行string的拼接
  object Test {
    def main(args: Array[String]): Unit = {
      var name = "zhangsan"
      var age = 14
      println(s"---hello world $name ${age + 4} ---")
    }
  }
  
  // 以上代码输出结果
  // ---hello world zhangsan 18 --- 

- class的个性化构造（var成员变量）

  ```scala
  object Test {
    def main(args: Array[String]): Unit = {
      var people = new People("zhangsan")
      people.printMsg()
    }
  }
  
  // 因为object体内的代码语句，是默认构造语句，当有需要传参的个性化构造时，必须要调用一下默认构造方法
  class People {
    var name = ""
    def this(_name: String) {
      this()
      name = _name
    }
  
    def printMsg(): Unit = {
      println(s"---$name---")
    }
  }
  
  // 以上代码输出结果
  // ---zhangsan---
  ```

- class的个性化构造（val成员变量）

  ```scala
  object Test {
    def main(args: Array[String]): Unit = {
      var people = new People(14)
      people.printMsg()
    }
  }
  
  class People(age: Int) {
  
    def printMsg(): Unit = {
      println(s"---$age---")
    }
  }
  
  // 以上代码输出结果
  // ---14---
  ```

- 伴生

  ```scala
  object Test {
  
    private val name = "bs"
  
    private var age = 10
  
    def main(args: Array[String]): Unit = {
      var test = new Test()
      test.printMsg()
    }
  
    def setName : Unit = {
      age = age + 14
    }
  }
  
  class Test() {
  
    def printMsg(): Unit = {
  
      Test.setName
      println(s"---${Test.name} ${Test.age}---")
    }
  }
  
  // 当object名称与class名称一致时，class名称可以调用object的变量和方法
  // 以上代码输出结果
  // ---bs 24---
  
  object Test {
  
    private val name = "object:name"
  
    def main(args: Array[String]): Unit = {
      var test = new Test()
      test.printMsg()
    }
  }
  
  class Test() {
  
    private val name = "class:name"
  
    def printMsg(): Unit = {
      println(s"---${Test.name}---")
    }
  }
  
  // 当object和class的成员变量一致时，输出的结果会是object类的成员变量
  // 以上代码输出结果
  // ---object:name---
  ```

  



































