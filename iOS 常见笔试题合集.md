1.  MVC 和MVVM 的区别？

   MVC: 分为Model-View-Controller，Model：负责数据模型， View：负责显示视图， Controller：负责协调用户事件，更新UI、更新模型。它的主要缺点：Controller过于臃肿，所有的UI逻辑以及业务逻辑，都集中与Controller中。

   MVVM：是MVC的一种改进，为了分离业务逻辑代码。分为：Model：数据模型，View: 视图显示，包括（view、controller），ViewModel：负责业务逻辑处理，协调view和model。实现了一种双向绑定机制。

   优点：降低了view-model的代码耦合。

   缺点：增加了胶水代码。

2. #import 和 #include的区别， #import“”和#import<>的区别？

   #import是OC中引入头文件的方式，#include是c/c++中引入头文件的方式，#import保证了只会引入一次头文件，不会重复；#import“” 是引入用户自定义的头文件，#import<> 是引入系统库头文件，或者自定义库头文件，或者pod库头文件。

3. frame 和 bounds 有什么区别？

   frame： 是view在父view坐标系统中的位置和大小；

   bounds：是view在自身系统中的位置和大小；

4. Objective-C 的类可以多继承么？没有的话什么可以代替？重写一个类用category好还是继承好？为什么？

   Objective-C中不可以多继承，但是我们可以用伪多继承的方式来实现，其实现方式为protocol和category、消息转发机制；重写一个类用分类好还是用继承好，需要看需求， 不同的需求下，用继承和分类有不同的好处。

5. @property 的本质是什么？ivar、getter、setter 是如何生成并添加到这个类中的？

   @property的本质就是：ivar+setter+getter；ivar 是编译器通过在属性名前加_自动合成实例变量，添加到类的实例变量列表中的，存取方法，也是编译器，主动生成的代码。

6. @property 中有哪些属性关键字以及作用？

   nonatomic：非原子操作。决定编译器生成的getter、setter方法是否是原子操作。

   atomic：存取原子性操作，效率较低。

   strong：持有特性，setter 方法对传入的参数先保留，后复制；

   weak：弱引用，setter 

7. 

   

   

   

   