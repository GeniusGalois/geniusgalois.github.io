---
title: "OC二进制文件MachO静态结构分析"
date: 2021-12-28T21:41:42+01:00
draft: false
tags: ["CSAPP","OS"]
summary: "OC二进制文件MachO静态结构分析"
---

自文件偏移量0开始，二进制文件内存结构如下：

## MachO Header（头部）

用于快速确认该文件的CPU类型、文件类型等信息

## LoadCommands（加载命令）

用于告诉loader如何设置并加载二进制数据

## Data （数据段 segment）

存放数据：代码、字符常量、类、方法等
可以拥有多个segment，每个segment可以有零到多个section，每个段都有一段虚拟地址映射到进程的地址空间，数据段是重要的部分，详细展开这段如下。

### __TEXT Segment （Readable/Executable）

#### __text Section

可执行文件的代码区 方法与函数的实现按照代码定义顺序存储在这里

1. 类的成员函数

   - 自定义的类方法和实例方法与纯c成员函数
   - 编译器生成的getter/setter方法
   - 编译器生成的.cxx_destruct方法（ARC环境）  
   > .cxx_destruct方法在ARC环境下由编译器生成实现类销毁时自动释放对象内存的工作，内部实现为对类拥有的所有对象(例如block)调用objc_storeStrong()方法release一次详细信息见《OC的MRC与ARC内存管理》

2. 全局纯c函数
3. main函数

#### __stubs Section

引用的链接库的函数的IMP，跳转到各个链接函数的地址

#### __stub_helper Section

#### __objc_classname Section

1. 各个类的struct __objc_class {.name}字段
2. 各个类的struct __objc_class {.ivarlayout .weakivarlayout}字段

#### __objc_methname Section

1. 各个类的方法的 struct __objc_method {.name}字段

   - 自定义的类方法和实例方法
   - 编译器生成的getter/setter方法
   - 编译器生成的.cxx_destruct方法（ARC环境）

2. 各个类的Ivar名（非属性和属性自动生成的）struct __objc_ivar {.name}字段 
3. 各个类的属性的attributes struct __objc_property {.attributes}字段

**OC类型编码（TypeEncodings、Property Attribute、Method Type）**
TypeEncodings
类型编码，可以用@encode(类型)获得，如@encode(typeof(变量))或者@encode(类型)

|  类型  | 编码 |
|  ----  | ----  |
| 基本数据类型  如int | i |
| char * | * |
| Class | # |
| A method selector (SEL) | : |
| An unknown type (among other things, this code is used for function pointers) | ? |
| A pointer to type | ^type |
| 对象  如id | @ |
| block | @? |
| 对象类型  如NSString | @"NSString" |
| 函数指针IM  如@property int (*functionPointerDefault)(char *) | ^? |
| struct和union  如struct stu{int a;NSString *b;}; | {struct名=类型1类型2} {stu=i@} |
| 数组如int a[10] | [10i] |

Property Attribute Description
属性的attributes格式：T类型，关键字，V属性名，默认关键字atomic retain/strong readwrite可省略。

|  属性  | attributes |
|  ----  | ----  |
| 基本数据类型  如@property char charDefault | Tc,V_charDefault |
| 对象类型  如@property(nonatomic, readonly, copy) id idA | T@,R,N,C,V_idA |
| 自定义getter/setter  如@property(getter=intGetFoo, setter=intSetFoo:) int intSetterGetter  | Ti,GintGetFoo,SintSetFoo:,V_intSetterGetter |
| @property (nonatomic,assign)  如struct stu *ps | T^{stu=i@},N,V_ps |

Method Type
函数签名的格式：返回类型+参数总长度+（参数类型+id）+（参数类型+参数偏移量）*n个参数
Method隐式的包含前两个参数 id self，SEL _cmd，所以签名参数前两个是（@0:8)
|  函数  | 签名 |
|  ----  | ----  |
| +(NSMutableArray *)fun0:(NSMutableDictionary *)dict | @24@0:8@16 |
| +(int)Inta | i16@0:8 |




#### __objc_methtype Section（方法签名）

1.各个类的方法的签名 struct __objc_method {.type)字段，不同方法相同sign会共用

   - 自定义的类方法和实例方法
   - 编译器生成的getter/setter方法
   - 编译器生成的.cxx_destruct方法（ARC环境）

2.各个类的Ivar类型编码（非属性和属性自动生成的）struct __objc_ivar {.type}

#### __cstring Section
字符常量区，存类C 风格的字符串

### __DATA_CONST Segment （Readable/Writable）

#### __got Section

#### __cfstring Section

对__cstring Section的引用，对外接口

#### __objc_imageinfo Section

### __DATA Segment （Readable/Writable）

#### __la_symbol_ptr Section

懒加载指针表，Section __stubs跳转的位置

####  __objc_const Section

1. struct __objc_method_list {.flags .method count .list包含的所有struct __objc_method{.name .type .implementation}}
2. struct __objc_ivar_list {.entsize .count .list包含的所有struct __objc_ivar{.offset(指向Section __objc_ivar) .name .type .size}}
3. struct __objc_property_list {.entsize .count .list包含的所有struct  __objc_property{.name .attributes}}
4. struct __objc_protocol_list {.entsize .count .list包含的所有struct  __objc_protocol{.name .attributes}}
5.  __OBJC_METACLASS_RO_$_classA和__OBJC_CLASS_RO_$_classA。Section __objc_data里 struct __objc_class {.data}指向的位置

``` 
                     __OBJC_METACLASS_RO_$_classA:
0000000100008100         struct __objc_data {                                   ; "classA", DATA XREF=_OBJC_METACLASS_$_classA
                             0x185,                               // flags
                             40,                                  // instance start
                             40,                                  // instance size
                             0x0,
                             0x0,                                 // ivar layout
                             aClassa,                             // name
                             __OBJC_$_CLASS_METHODS_classA,       // base methods
                             __OBJC_CLASS_PROTOCOLS_$_classA,     // base protocols
                             0x0,                                 // ivars
                             0x0,                                 // weak ivar layout
                             0x0                                  // base properties
                         }
                             __OBJC_CLASS_RO_$_classA:
0000000100008350         struct __objc_data {                               ; "classA","S", DATA XREF=_OBJC_CLASS_$_classA
                             0x184,                               // flags
                             8,                                   // instance start
                             72,                                  // instance size
                             0x0,
                             aS,                                  // ivar layout
                             aClassa,                             // name
                             __OBJC_$_INSTANCE_METHODS_classA,    // base methods
                             0x0,                                 // base protocols
                             __OBJC_$_INSTANCE_VARIABLES_classA,  // ivars
                             0x0,                                 // weak ivar layout
                             __OBJC_$_PROP_LIST_classA            // base properties
                         }
```

####  __objc_classrefs Section

class对代码段的接口地址，指向Section __objc_data的存着的class

####  __objc_ivar Section
ivar的存储空间，Section __objc_const里struct __objc_ivar {.offset pointer }指向的位置

####  __objc_data Section

class和metaclass的定义位置

```
                             _OBJC_METACLASS_$_classA:
00000001000083e0         struct __objc_class {                                  ; DATA XREF=_OBJC_CLASS_$_classA
                             _OBJC_METACLASS_$_NSObject,          // metaclass
                             _OBJC_METACLASS_$_NSObject,          // superclass
                             __objc_empty_cache,                  // cache
                             0x0,                                 // vtable
                             __OBJC_METACLASS_RO_$_classA         // data
                         }

                             _OBJC_CLASS_$_classA:
0000000100008408         struct __objc_class {                                  ; DATA XREF=0x100004068, objc_cls_ref_classA
                             _OBJC_METACLASS_$_classA,            // metaclass
                             _OBJC_CLASS_$_NSObject,              // superclass
                             __objc_empty_cache,                  // cache
                             0x0,                                 // vtable
                             __OBJC_CLASS_RO_$_classA             // data
                         }
```

##  Loader Info （链接信息）
一个完整的用户级MachO文件的末端是一系列链接信息。其中包含了动态加载器用来链接可执行文件或者依赖所需使用的符号表、字符串表等


__DATA Segment 待补充的 Section:
__nl_symbol_ptr: 非懒加载指针表,dyld 加载会立即绑定
__ls_symbol_ptr: 懒加载指针表
__mod_init_func: constructor 函数
__mod_term_func: destructor 函数
__objc_classlist: 类列表
__objc_nlclslist: 实现了 load 方法的类
__objc_protolist: protocol的列表
__objc_classrefs: 被引用的类列表
__objc _catlist: Category列表
