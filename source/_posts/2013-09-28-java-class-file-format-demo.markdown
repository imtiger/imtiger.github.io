---
layout: post
title: "实例分析Java Class文件的结构-实践篇"
date: 2013-01-30 12:16
comments: true
categories: 
- Java
- 技术
---
{% img center /images/2013/01/30/java-logo.jpg 500 300 %}  
在[`实例分析Java Class文件的结构-理论篇`](/blog/2013/01/30/java-class-file-format/)一文中，我们说了Java Class文件结构的理论知识，接下来我们来通过一个具体的例子来理论结合实践的学习一下。
首先我们有一个TestClass类，代码如下：
```java TestClass.java 
     package com.ejushang.TestClass;
     public class TestClass implements Super{
     
             private static final int staticVar = 0;

             private int instanceVar=0;
     
             public int instanceMethod(int param){
                 return param+1;
             }
              
     }

     interface Super{ }
```

通过jdk1.6.0_37的javac 编译后的TestClass.java对应的TestClass.class的二进制结构如下图所示：
<!-- more -->

{% img center /images/2013/01/30/test-class-file.png %}  

1. 魔数  
从Class的文件结构我们知道，刚开始的4个字节是魔数，上图中从地址00000000h-00000003h的内容就是魔数，从上图可知Class的文件的魔数是0xCAFEBABE。

2. 主次版本号   
接下来的4个字节是主次版本号，由上图可知从00000004h-00000005h对应的是0x0000,因此Class的minor_version为0x0000,从00000006h-00000007h对应的内容为0x0032,因此Class文件的major_version版本为0x0032,这正好就是jdk1.6.0不带target参数编译后的Class对应的主次版本。

3.  常量池的数量   
接下来的2个字节从00000008h-00000009h表示常量池的数量，由上图可以知道其值为0x0018，十进制为24个,但是对于常量池的数量需要明确一点，常量池的数量是constant_pool_count-1，为什么减一，是因为索引0表示class中的数据项不引用任何常量池中的常量。

4.  常量池
我们上面说了常量池中有不同类型的常量，下面就来看看TestClass.class的第一个常量，我们知道每个常量都有一个u1类型的tag标识来表示常量的类型，上图中0000000ah处的内容为0x0A，转换成二级制是10，由上面的关于常量类型的描述可知tag为10的常量是Constant_Methodref_info,而Constant_Methodref_info的结够如下图所示：
{% img center /images/2013/01/30/constant-methodref-info.png %}   
`class_index`指向常量池中类型为CONSTANT_Class_info的常量，从TestClass的二进制文件结构中可以看出class_index的值为0x0004（地址为0000000bh-0000000ch)，也就是说指向第四个常量。   
`name_and_type_index`指向常量池中类型为CONSTANT_NameAndType_info常量。从上图可以看出name_and_type_index的值为0x0013，表示指向常量池中的第19个常量.接下来又可以通过同样的方法来找到常量池中的所有常量。不过JDK提供了一个方便的工具可以让我们查看常量池中所包含的常量。通过javap -verbose TestClass 即可得到所有常量池中的常量，截图如下：  
{% img center /images/2013/01/30/test-class-javap.png %}   
从上图我们可以清楚的看到，TestClass中常量池有24个常量，不要忘记了第0个常量，因为第0个常量被用来表示 Class中的数据项不引用任何常量池中的常量。从上面的分析中我们得知TestClass的第一个常量表示方法，其中class_index指向的第四个常量为java/lang/Object，name_and_type_index指向的第19个常量值为<init>:()V,从这里可以看出第一个表示方法的常量表示的是java编译器生成的实例构造器方法。通过同样的方法可以分析常量池的其它常量。OK，分析完常量池，我们接下来再分析下access_flags。  
5. u2 access_flags 
表示类或者接口方面的访问信息，比如Class表示的是类还是接口，是否为public,static，final等。具体访问标示的含义之前已经说过了，下面我们就来看看TestClass的访问标示。Class的访问标示是从0000010dh-0000010e，其值为0x0021，根据前面说的各种访问标示的标志位，我们可以知道：0x0021=0x0001|0x0020 也即ACC_PUBLIC 和 ACC_SUPER为真，其中ACC_PUBLIC大家好理解，ACC_SUPER是jdk1.2之后编译的类都会带有的标志。
6. u2 this_class   
表示类的索引值，用来表示类的全限定名称，类的索引值如下图所示：  
{% img center  /images/2013/01/30/class-index.png %}  
从上图可以清楚到看到，类索引值为0x0003，对应常量池的第三个常量，通过javap的结果，我们知道第三个常量为CONSTANT_Class_info类型的常量，通过它可以知道类的全限定名称为：com/ejushang/TestClass/TestClass

7. u2 super_class   
表示当前类的父类的索引值，索引值指向常量池中类型为CONSTANT_Class_info的常量，父类的索引值如下图所示，其值为0x0004,查看常量池的第四个常量，可知TestClass的父类的全限定名称为：java/lang/Object.
{% img center  /images/2013/01/30/super-class.png %}  

8. interfaces_count 、 interfaces[interfaces_count]  
表示接口数量以及具体的每一个接口，TestClass的接口数量以及接口如下图所示，其中0x0001表示接口数量为1，而0x0005表示接口在常量池的索引值，找到常量池的第五个常量，其类型为CONSTANT_Class_info，其值为：com/ejushang/TestClass/Super
{% img center /images/2013/01/30/interface-count.png %}

9. fields_count、field_info   
fields_count表示类中field_info表的数量，而field_info表示类的实例变量和类变量，这里需要注意的是field_info不包含从父类继承过来的字段，field_info的结构如下图所示：  
{% img center /images/2013/01/30/field-info.png %}  
`access_flags`表示字段的访问标识，比如public,private,protected，static,final等，access_flags的取值如下图所示：  
{% img center /images/2013/01/30/access-flags.png %}  
`name_index` 和 `descriptor_index`都是常量池的索引值，分别表示字段的名称和字段的描述符，字段的名称容易理解，但是字段的描述符如何理解呢？其实在JVM 规范中，对于字段的描述符规定如下图所示：  
{% img center /images/2013/01/30/field-descriptor.png %}  
其中大家需要关注一下上图最后一行，它表示的是对一维数组的描述符，对于String[][]的描述符将是[[ Ljava/lang/String,而对于int[][]的描述符为[[I。接下来的attributes_count以及attribute_info分别表示属性表的数量以及属性表。  
下面我们还是以上面的TestClass为例，来看看TestClass的字段表吧。首先我们来看一下字段的数量，TestClass的字段的数量如下图所示：  
{% img center /images/2013/01/30/testclass-field-count.png %}  
从上图中可以看出TestClass有两个字段，查看TestClass的源代码可知，确实也只有两个字段，接下来我们看看第一个字段，我们知道第一个字段应该为private int staticVar,它在Class文件中的二进制表示如下图所示：  
{% img center /images/2013/01/30/field-one.png %}  
其中0x001A表示访问标示，通过查看access_flags表可知，其为ACC_PRIVATE,ACC_STATIC,ACC_FINAL,接下来0x0006和0x0007分别表示常量池中第6和第7个常量，通过查看常量池可知，其值分别为：staticVar和I，其中staticVar为字段名称，而I为字段的描述符，通过上面对描述符的解释，I所描述的是int类型的变量，接下来0x0001表示staticVar这个字段表中的属性表的数量，从上图可以staticVar字段对应的属性表有1个，0x0008表示常量池中的第8个常量，查看常量池可以得知此属性为ConstantValue属性，而ConstantValue属性的格式如下图所示：  
{%img center /images/2013/01/30/constantvalue-attribute.png%}  
其中`attribute_name_index`表述属性名的常量池索引，本例中为ConstantValue，而ConstantValue的`attribute_length`固定长度为2，而`constantValue_index`表示常量池中的引用，本例中，其中为0x0009，查看第9个常量可以知道，它表示一个类型为CONSTANT_Integer_info的常量，其值为0。  
上面说完了`private static final int staticVar=0`，下面我们接着说一下TestClass的`private int instanceVar=0`,在本例中对instanceVar的二进制表示如下图所示：
{% img center /images/2013/01/30/field-two.png %}  
其中0x0002表示访问标示为ACC_PRIVATE,0x000A表示字段的名称，它指向常量池中的第10个常量，查看常量池可以知道字段名称为instanceVar，而0x0007表示字段的描述符，它指向常量池中的第7个常量，查看常量池可以知道第7个常量为I，表示类型为instanceVar的类型为I，最后0x0000表示属性表的数量为0.
10. methods_count 、method_info   
methods_count表示方法的数量，而method_info表示的方法表，其中方法表的结构如下图所示：  
{% img center /images/2013/01/30/method-info.png %}    
从上图可以看出method_info和field_info的结构是很类似的，方法表的access_flag的所有标志位以及取值如下图所示：  
{% img center /images/2013/01/30/access-flags.png %}    
其中`name_index`和`descriptor_index`表示的是方法的名称和描述符，他们分别是指向常量池的索引。这里需要结解释一下方法的描述符，方法的描述符的结构为：`（参数列表）返回值`，比如`public int instanceMethod(int param)`的描述符为：`（I）I`，表示带有一个int类型参数且返回值也为int类型的方法，接下来就是属性数量以及属性表了，方法表和字段表虽然都有属性数量和属性表，但是他们里面所包含的属性是不同。接下来我们就以TestClass来看一下方法表的二进制表示。首先来看一下方法表数量，截图如下：
{% img center /images/2013/01/30/method-count.png %}  
从上图可以看出方法表的数量为0x0002表示有两个方法，接下来我们来分析第一个方法，我们首先来看一下TestClass的第一个方法的`access_flag`，`name_index`,`descriptor_index`，截图如下：  
{% img center /images/2013/01/30/method-one-info.png %}  
从上图可以知道access_flags为0x0001，从上面对access_flags标志位的描述，可知方法的access_flags的取值为ACC_PUBLIC,name_index为0x000B，查看常量池中的第11个常量，知道方法的名称为<init>，0x000C表示descriptor_index表示常量池中的第12常量，其值为()V,表示<init>方法没有参数和返回值，其实这是编译器自动生成的实例构造器方法。接下来的0x0001表示<init>方法的方法表有1个属性，属性截图如下：
{% img center /images/2013/01/30/init-method-info.png %}   
从上图可以看出0x000D对应的常量池中的常量为Code,表示的方法的Code属性，所以到这里大家应该明白方法的那些代码是存储在Class文件方法表中的属性表中的Code属性中。接下来我们在分析一下Code属性，Code属性的结构如下图所示：
{% img center /images/2013/01/30/code-attribute.png %}  
其中`attribute_name_index`指向常量池中值为Code的常量，`attribute_length`的长度表示Code属性表的长度（这里需要注意的时候长度不包括attribute_name_index和attribute_length的6个字节的长度）。  
`max_stack`表示最大栈深度，虚拟机在运行时根据这个值来分配栈帧中操作数的深度.  
`max_locals`代表了局部变量表的存储空间,它的单位为slot，slot是虚拟机为局部变量分配内存的最小单元，在运行时，对于不超过32位类型的数据类型，比如byte,char,int等占用1个slot，而double和Long这种64位的数据类型则需要分配2个slot，另外max_locals的值并不是所有局部变量所需要的内存数量之和，因为slot是可以重用的，当局部变量超过了它的作用域以后，局部变量所占用的slot就会被重用。   
`code_length`代表了字节码指令的数量，而code表示的时候字节码指令，从上图可以知道code的类型为u1,一个u1类型的取值为0x00-0xFF,对应的十进制为0-255，目前虚拟机规范已经定义了200多条指令。  
`exception_table_length`以及`exception_table`分别代表方法对应的异常信息。  
 `attributes_count`和`attribute_info`分别表示了Code属性中的属性数量和属性表，从这里可以看出Class的文件结构中，属性表是很灵活的，它可以存在于Class文件，方法表，字段表以及Code属性中。
接下来我们继续以上面的例子来分析一下，从上面init方法的Code属性的截图中可以看出，属性表的长度为0x00000026,max_stack的值为0x0002,max_locals的取值为0x0001,code_length的长度为0x0000000A，那么00000149h-00000152h为字节码，接下来exception_table_length的长度为0x0000，而attribute_count的值为0x0001，00000157h-00000158h的值为0x000E,它表示常量池中属性的名称，查看常量池得知第14个常量的值为LineNumberTable，LineNumberTable用于描述java源代码的行号和字节码行号的对应关系，它不是运行时必需的属性,接下来我们再看一下LineNumberTable的结构如下图所示：

>如果通过-g:none的编译器参数来取消生成LineNumberTable的话，最大的影响就是异常发生的时候，堆栈中不能显示出出错的行号，调试的时候也不能按照源代码来设置断点，

{% img center /images/2013/01/30/linenumbertable-attribute.png %}  
其中`attribute_name_index`上面已经提到过，表示常量池的索引，`attribute_length`表示属性长度，而`start_pc`和`line_number`分表表示字节码的行号和源代码的行号。本例中LineNumberTable属性的字节流如下图所示：
{% img center /images/2013/01/30/testclass-linenumbertable.png %}  
上面分析完了TestClass的第一个方法，通过同样的方式我们可以分析出TestClass的第二个方法，截图如下：  
{%img center /images/2013/01/30/method-two.png %}
其中`access_flags`为0x0001,`name_index`为0x000F,`descriptor_index`为0x0010，通过查看常量池可以知道此方法为`public int instanceMethod(int param)`方法。通过和上面类似的方法我们可以知道instanceMethod的Code属性为下图所示：  
{% img center /images/2013/01/30/method-two-code-attribute.png %}     

最后我们来分析一下，Class文件的属性，从00000191h-00000199h为Class文件中的属性表，其中0x0011表示属性的名称，查看常量池可以知道属性名称为SourceFile，我们再来看看SourceFile的结构如下图所示：      
{% img center /images/2013/01/30/sourcefile-attribute.png %}  
其中`attribute_length`为属性的长度，`sourcefile_index`指向常量池中值为源代码文件名称的常量，在本例中SourceFile属性截图如下：
{% img center /images/2013/01/30/test-class-sourcefile.png %}  
其中`attribute_length`为0x00000002表示长度为2个字节，而`soucefile_index`的值为0x0012,查看常量池的第18个常量可以知道源代码文件的名称为TestClass.java

通过[`实例分析Java Class文件的结构-理论篇`](/blog/2013/01/30/java-class-file-format/)和[`实例分析Java Class文件的结构-实践篇`](/blog/2013/01/30/java-class-file-format-demo/)两篇文章，我们采用理论和实践结合方式来学习了Class 文件的格式。掌握它的格式以后，我们也可以试着写个Java Class类文件的反编译器了。


