# Category

### Category内部实现</br>

```objc
//分类的结构:
struct _category_t {
    const char *name;//分类所属的类名
    struct _class_t *cls;
    const struct _method_list_t *instance_methods;
    const struct _method_list_t *class_methods;
    const struct _protocol_list_t *protocols;
    const struct _prop_list_t *properties;
}
```
程序一编译,分类的信息都会存储在_category_t这个结构体下,相当于编写出一个分类,就生成了一个对应的结构体对象,例如有5个分类,到时就会生成对应5个结构体对象,每个结构体变量的变量名和内部存储的值是不同的,此时并没有合并到类对象对应的方法中

* 误区:一个类值存在一个类对象,所以不存在说创建一个分类就创建一个对应分类的对象这么一个情况
* 分类中的方法合并:通过runtime动态将分类的方法合并到类对象、元类对象中

>分类的内部实现过程:</br>
>1.通过runtime加载某个类的所有category数据</br>
>2.把所有category的方法、属性、协议数据合并到一个大数组中(其中,后面参与编译的category数据,会在数组的前面)</br>
>3.将合并后的分类数据(方法、属性、协议),插入到原来数据的前面</br>
>>* 由源码可以得出,最后面编译的分类,在方法列表中位置最靠前
>>* 在xcode中Build Phases -  Compile Sources中可以手动设置编译的顺序

category(分类)与extension(类扩展)的区别:
* 类扩展内容一编译就已经合并到类中了,分类中是在运行时通过runtime合并到类信息中

category的实现原理:
* category编译之后的底层结构是struct category_t,里面存储着分类的对象方法、类方法、属性、协议信息
* 在程序运行的时候,runtime会将category的数据,合并到类信息中(类对象、元类对象中)

通过查看源码,发现底层将category插入数组时,调用两个函数`memmove`和`memcpy`,二者的区别是:</br>
memmove:会先判断是往左挪还是往右挪(会保证完整性)</br>
memcpy:会一个一个拷贝(从小地址开始)
>例如存在下面4块内存空间 1  2  3  4</br>
>现在要将1和2,移动到2和3的位置,通过memmove可以实现3 1 2 4</br>
>通过memcpy实现即为:1 1 1 4

### +load()方法</br>
* 会在runtime加载类、分类时调用
* 每个类、分类的+load,在程序运行过程中只调用一次

从源码分析,会先调用类的load方法,再调用分类的load方法</br>
之所以运行时,类调用load方法和分类调用load方法不会像其他方法一样被覆盖,是因为内部实现中,会直接取出load方法的函数地址直接进行调用,而不是通过消息转发(objc_msgSend)

>如果是通过消息机制调用方法,流程都是通过isa指针,找到对应的类方法/元类方法,在方法列表中按顺序查找</br>
而load方法是直接通过load方法在内存中的地址值直接调用的

* 调用顺序:</br>
  1.先调用类的load方法</br>
  2.按照编译的先后顺序调用(先编译,先调用)</br>
  3.调用子类的load方法之前会先调用父类的load方法</br>
  4.调用分类的load方法</br>
  5.按照编译的先后顺序调用(先编译,先调用)</br>

一道面试题:</br>
>Category中有load方法吗?load方法是什么时候调用的?load方法能继承吗?</br>
>* 有load方法
>* load方法在runtime加载类、分类的时候调用
>* load方法可以继承,但是一般情况下不会主动去调用load方法,都是系统自动调用

### +initialize()方法</br>
* 会在类第一次接收到消息时调用

* 调用顺序:</br>
  1.先调用父类的+initialize,再调用子类的+initialize(前提是父类之前没有被初始化过,)</br>
  2.先初始化父类,再初始化子类,每个类只会初始化一次

>之所以调用子类的方法,会调用父类的initialize方法,是因为内部主动发送了2条消息(一条是给父类发送initialize消息,一条是给子类发送initialize消息)</br>
>注:initialize方法最终是通过objc_msgSend方法调用的

+initialize()和+load()的很大区别是,+initialize是通过objc_msgSend进行调用的,所以有以下特点:</br>
* 如果子类没有实现+initialize,会调用父类的+initialize方法(所以父类的+initialize方法可能会被调用多次)
* 如果分类实现了+initialize,就覆盖了类本身的+initialize调用

### +load()方法和+initialize()方法总结</br>
1.调用方式</br>
* load是根据函数地址直接调用
* initialize是通过objc_msgSend调用</br>

2.调用时刻</br>
* load是runtime加载类、分类的时候调用(只会调用一次)
* initialize是类第一次接收到消息的时候调用,每一个类只会initialize一次,但父类的initialize方法可能会被调用多次</br>

3.调用顺序</br>

　　1)load:</br>
* 先调用类的load 1)先编译的类优先调用load 2)调用子类的load之前,会先调用父类的load 
* 再调用分类的load 1)先编译的分类,优先调用load</br>

　　2)initialize:</br>
* 先初始化父类
* 再初始化子类(可能最终调用的是父类的initialize方法)

### 关联对象</br>
```objc
@property (nonatomic, assign)int age;
```
在类中声明属性,系统会做3件事:</br>
1.生成带下划线成员变量 </br>
2.setter和getter声明 </br>
3.setter和getter的实现 </br>

分类中添加属性,则只会生成getter和setter的声明

要想与类一样,实现一个属性的读写,可供解决方案有:</br>
1.通过全局变量当中间值赋值,问题是没次创建一个属性都是共用一个全局变量,会导致数据错乱</br>
2.使用可变字典作为全局变量,问题:1.存在线程安全的问题2.每次添加属性比较麻烦

>默认情况下,因为分类底层结构的限制,不能添加成员变量到分类中,但可以通过关联对象来间接实现

```objc
//添加关联对象
void objc_setAssociatedObject(id  _Nonnull object, const void * _Nonnull key, 
                              id  _Nullable value, objc_AssociationPolicy policy);
                              
//获得关联对象
id objc_getAssociatedObject(id  _Nonnull object, const void * _Nonnull key)

//移除所有关联对象
void objc_removeAssociatedObjects(id  _Nonnull object)
```

* 关联策略policy</br>
![](Snip20180703_24.png)

* 关键字key</br>
关联属性方法中的key获取方式:</br>
1.可以直接通过const void *name = &name方式,即传入name获取</br>
2.static const char name.传入&name获取</br>
3.key处传入@selector(name);</br>
4.仅限于getter,直接在参数处传入_cmd</br>
  
  ```objc
  objc_getAssociatedObject(self, _cmd)
  ```

>注:</br>
>　1)往全局变量添加一个static,令该全局变量作用域仅当前文件</br>
>　2)直接将字符串写入,由于字符串位于常量区,故所有的值都是相同的</br>
>　3)因为每个方法其实都默认传递2个参数,第一个是self,第二个是_cmd,故也可以使用一下方式传递key值,setter方法不行,因为两者的@selector是不同的</br>
>　4)关联对象中存入的值既不是存放类对象中,也不是存放在实例对象中

实现关联对象技术的核心对象有:</br>
* AssociationsManager
* AssociationHashMap
* ObjectAssociationMap
* ObjcAssociation

![](Snip20180703_30.png)

其中的disguised_ptr_t是经过一个DISGUISE函数(其内部就是获取object地址进行相应的位运算)

>若查找时发现value已经销毁或不存在,则系统会报坏内存访问,因为ObjcAssociation内部对value是强引用的