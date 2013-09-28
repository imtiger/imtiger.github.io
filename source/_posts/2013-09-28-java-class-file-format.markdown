---
layout: post
title: "实例分析Java Class文件的结构-理论篇"
date: 2013-01-30 10:33
comments: true
categories: 
- Java
- 技术
---
今天把之前在Evernote中的笔记重新整理了一下，发上来供对java class 文件结构的有兴趣的同学参考一下。

学习Java的朋友应该都知道Java从刚开始的时候就打着平台无关性的旗号，说“一次编写，到处运行”，其实说到无关性，Java平台还有另外一个无关性那就是语言无关性，要实现语言无关性，那么Java体系中的class的文件结构或者说是字节码就显得相当重要了，其实Java从刚开始的时候就有两套规范，一个是Java语言规范，另外一个是Java虚拟机规范，Java语言规范只是规定了Java语言相关的约束以及规则，而虚拟机规范则才是真正从跨平台的角度去设计的。今天我们就以一个实际的例子来看看，到底Java中一个Class文件对应的字节码应该是什么样子。 这篇文章将首先总体上阐述一下Class到底由哪些内容构成，然后再用一个实际的Java类入手去分析class的文件结构。 

<!-- more --> 

在继续之前，我们首先需要明确如下几点：  
1. Class文件是有8位为基础的字节流构成的，这些字节流之间都严格按照规定的顺序排列，并且字节之间不存在任何空隙，对于超过8位的数据，将按照Big-Endian的顺序存储的，也就是说高位字节存储在低的地址上面，而低位字节存储到高地址上面，其实这也是class文件要跨平台的关键，因为PowerPC架构的处理采用Big-Endian的存储顺序，而x86系列的处理器则采用Little-Endian的存储顺序，因此为了Class文件在各中处理器架构下保持统一的存储顺序，虚拟机规范必须对起进行统一。      
2. Class文件结构采用类似C语言的结构体来存储数据的，主要有两类数据项，无符号数和表，无符号数用来表述数字，索引引用以及字符串等，比如u1,u2,u4,u8分别代表1个字节，2个字节，4个字节，8个字节的无符号数，而表是有多个无符号数以及其它的表组成的复合结构。可能大家看到这里对无符号数和表到底是上面也不是很清楚，不过不要紧，等下面实例的时候，我会再以实例来解释。   
     
明确了上面的两点以后，我们接下来后来看看Class文件中按照严格的顺序排列的字节流都具体包含些什么数据：  
{% img center /images/2013/01/30/java-class-file-format.png %}  

  <center>（上图来自The Java Virtual Machine Specification Java SE 7 Edition) </center>    

在看上图的时候，有一点我们需要注意，比如cp_info，cp_info表示常量池，上图中用constant_pool[constant_pool_count-1]的方式来表示常量池有constant_pool_count-1个常量，它这里是采用数组的表现形式，但是大家不要误以为所有的常量池的常量长度都是一样的，其实这个地方只是为了方便描述采用了数组的方式，但是这里并不像编程语言那里，一个int型的数组，每个int长度都一样.明确了这一点以后，我们再回过头来看看上图中每一项都具体代表了什么含义。  

1. u4 magic 表示魔数，并且魔数占用了4个字节，魔数到底是做什么的呢？它其实就是表示一下这个文件的类型是一个Class文件，而不是一张JPG图片，或者AVI的电影。而Class文件对应的魔数是0xCAFEBABE.  
2. u2 minor_version 表示Class文件的次版本号，并且此版本号是u2类型的无符号数表示。   
3. u2 major_version 表示Class文件的主版本号，并且主版本号是u2类型的无符号数表示。major_version和minor_version主要用来表示当前的虚拟机是否接受当前这种版本的Class文件。不同版本的Java编译器编译的Class文件对应的版本是不一样的。高版本的虚拟机支持低版本的编译器编译的Class文件结构。比如Java SE 6.0对应的虚拟机支持Java SE 5.0的编译器编译的Class文件结构，反之则不行。  
4. u2 constant_pool_count 表示常量池的数量。这里我们需要重点来说一下常量池是什么东西，请大家不要与Jvm内存模型中的运行时常量池混淆了，Class文件中常量池主要存储了字面量以及符号引用，其中字面量主要包括字符串，final常量的值或者某个属性的初始值等等，而符号引用主要存储类和接口的全限定名称，字段的名称以及描述符，方法的名称以及描述符，这里名称可能大家都容易理解，至于描述符的概念，放到下面说字段表以及方法表的时候再说。另外大家都知道Jvm的内存模型中有堆，栈，方法区，程序计数器构成，而方法区中又存在一块区域叫运行时常量池，运行时常量池中存放的东西其实也就是编译器产生的各种字面量以及符号引用，只不过运行时常量池具有动态性，它可以在运行的时候向其中增加其它的常量进去，最具代表性的就是String的intern方法。  
5. cp_info 表示常量池，这里面就存储了上面说的各种各样的字面量和符号引用。放到常量池的中数据项在`The Java Virtual Machine Specification Java SE 7 Edition` 中一共有14个常量，每一种常量都是一个表，并且每种常量都用一个公共的部分tag来表示是哪种类型的常量。下面分别简单描述一下,具体细节等到后面的实例中我们再细化。   
CONSTANT_Utf8_info      tag标志位为1,   UTF-8编码的字符串    
CONSTANT_Integer_info  tag标志位为3， 整形字面量         
CONSTANT_Float_info     tag标志位为4， 浮点型字面量      
CONSTANT_Long_info     tag标志位为5， 长整形字面量   
CONSTANT_Double_info  tag标志位为6， 双精度字面量   
CONSTANT_Class_info    tag标志位为7， 类或接口的符号引用  
CONSTANT_String_info    tag标志位为8，字符串类型的字面量  
CONSTANT_Fieldref_info  tag标志位为9,  字段的符号引用   
CONSTANT_Methodref_info  tag标志位为10，类中方法的符号引用  
CONSTANT_InterfaceMethodref_info tag标志位为11, 接口中方法的符号引用  
CONSTANT_NameAndType_info tag 标志位为12，字段和方法的名称以及类型的符号引用  
6. u2 access_flags 表示类或者接口的访问信息，具体如下图所示：
{% img center /images/2013/01/30/class-access-and-property-modifiers.png %}
7. u2 this_class 表示类的常量池索引，指向常量池中CONSTANT_Class_info的常量
8. u2 super_class 表示超类的索引，指向常量池中CONSTANT_Class_info的常量
9. u2 interface_counts 表示接口的数量
10. u2 interface[interface_counts]表示接口表，它里面每一项都指向常量池中CONSTANT_Class_info常量
11. u2 fields_count 表示类的实例变量和类变量的数量
12. field_info fields[fields_count]表示字段表的信息，其中字段表的结构如下图所示：  
{% img center /images/2013/01/30/field-info.png %}  
上图中access_flags表示字段的访问标示，比如字段是public,private，protect 等，name_index表示字段名 称，指向常量池中类型是CONSTANT_UTF8_info的常量，descriptor_index表示字段的描述符，它也指向常量池中类型为CONSTANT_UTF8_info的常量，attributes_count表示字段表中的属性表的数量，而属性表是则是一种用与描述字段，方法以及类的属性的可扩展的结构，不同版本的Java虚拟机所支持的属性表的数量是不同的。
13. u2 methods_count表示方法表的数量
14. method_info 表示方法表，方法表的具体结构如下图所示：
{% img center /images/2013/01/30/method-info.png %}  
其中access_flags表示方法的访问标示，name_index表示名称的索引，descriptor_index表示方法的描述符，attributes_count以及attribute_info类似字段表中的属性表，只不过字段表和方法表中属性表中的属性是不同的，比如方法表中就Code属性，表示方法的代码，而字段表中就没有Code属性。其中具体Class中到底有多少种属性，等到Class文件结构中的属性表的时候再说说。
15. attribute_count表示属性表的数量，说到属性表，我们需要明确以下几点：  
1.属性表存在于Class文件结构的最后，字段表，方法表以及Code属性中，也就是说属性表中也可以存在属性表。  
2.属性表的长度是不固定的，不同的属性，属性表的长度是不同的。 

本篇文章描述了Java Class文件方面的理论知识，下面一篇文章将通过一个实际的例子来详细解释一下Class文件内部到底长什么样。具体请参考本系列的第二篇文章：  
[实例分析Java Class文件的结构-实践篇](/blog/2013/01/30/java-class-file-format-demo/)








