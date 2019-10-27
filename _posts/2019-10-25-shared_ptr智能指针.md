# shared_ptr智能指针

[原文地址](https://www.tuicool.com/wx/QV3aMzY)

shared_ptr是通过指针保持对象共享所有权的智能指针。多个shared_ptr对象可占有同一资源，当最后一个shared_ptr对象被销毁或者通过operator=,reset()操作赋予另一指针时，其管理的资源才会被回收。
管理同一资源的不同shared_ptr对象能在不同线程中不加同步的调用其所有成员函数。当然这里指的是shared_ptr对象本身的成员函数，如果你想多线程访问其管理的资源，那么并不会有这种保证。

[其他成员变量详情](https://en.cppreference.com/w/cpp/memory/shared_ptr)

shared_ptr也可以指定删除器，但与unique_ptr不同的是，该删除器类型并不作为shared_ptr模板中的参数之一。
C++17之前，shared_ptr管理动态分配的数组需要提供自定义的删除器。c++17可以管理动态数组，例如shared_ptr<int[]> sp(new int[10])。为了支持这一点，element_type现在被定义为remove_extent_t<T>。
话不多说，我们来看它的源代码实现：

![源码实现](https://img1.tuicool.com/aIRrqeJ.png!mobile)

- 这个类没有成员变量，其数据存储于基类_Ptr_base<_Ty>中，在其成员函数中绝大部分操作也是调用基类提供的函数完成。

接下来我们分析一下_Ptr_base基类的实现。_Ptr_base是一个模板类，模板参数是shared_ptr管理的指针对应的类型。该基类拥有如下这两个数据成员：

![基类成员](https://img2.tuicool.com/7Vj2a2A.png!mobile)

关于element_type :

![element_type1](https://img1.tuicool.com/b2myQru.png!mobile)
![element_type2](https://img0.tuicool.com/R7n2quz.png!mobile)

- 可以看到element_type 就是模板参数对应的标量类型（scalar type）（since c++17）。

关于_Ref_count_base:

![ref_count_base](https://img2.tuicool.com/vEnumun.png!mobile)

- 从名字上就可以看出来，这是一个引用计数的基类，其有两个纯虚函数_Destroy()和_Delete_this()，并且拥有两个数据成员_Uses和_Weaks，其类型都为_Atomic_counter_t，shared_ptr之所以在多线程条件下可以不加同步的访问，就是因为其内部的引用计数的变化都是用原子操作实现的。这个基类还有对应的对_Uses和_Weaks操作的函数，这些函数保证了在多线程条件下修改引用计数的原子性，我们放到最后分析。


继承自该类的有五个类，我们研究前三个（后两个派生类目前没有发现是干嘛用的。。）：

![前三个派生类](https://img0.tuicool.com/iqeuUjj.png!mobile)

- 从模板参数可以看出来，这三个类分别代表了采用默认删除器和默认内存分配器的版本、采用自定义删除器和默认内存分配器的版本和采用自定义的删除器和内存分配器的版本。

我们先来分析较为常用的_Ref_count，即采用默认的删除器和内存分配器的版本的类：

![_Ref_count](https://img1.tuicool.com/qEvuayU.png!mobile)

- 这个类非常简洁，由于采用默认的删除和内存分配操作，因此只需要保留一个_Ty*类型的指针即可。

第二个派生类_Ref_count_resource:

![_Ref_count_resource](https://img0.tuicool.com/BrE7Znj.png!mobile)


- 这个类中我们见到了一个老朋友：_Compressed_pair。详见unique_ptr那一节中的讲解，这里不再赘述。

第三个派生类：


![_Ref_count_resource_alloc](https://img0.tuicool.com/NNfQf2Q.png!mobile)

- 可以看到，这里有一个嵌套的_Compressed_pair。因为删除器和内存分配器都很有可能是空类。

这些类具体的不同主要在两个虚函数中如何释放资源以及如何释放自己，这里不再赘述，应该很容易看懂。我们看一下这些派生类是什么时候生成并被赋值给_Ptr_base基类的_Rep成员的。跳回shared_ptr的构造函数：

![shared_ptr1](https://img2.tuicool.com/z6n2Yvr.png!mobile)
![shared_ptr2](https://img2.tuicool.com/EnqQreM.png!mobile)

- 以只接收一个指针_Ux*的构造函数为例，该函数内部调用了_Setp函数。这里的is_array<_Ty>{}生成了一个临时对象，大家在源代码里跳过去能看到这里是做了一个函数选择，利用函数重载的功能，根据_Ty是否是一个数组类型将其分发给对应的重载函数。这种技巧在stl库中应用的很多，具体名字被我给忘了。。。这里不多解释，接着看。

![_step_true](https://img0.tuicool.com/uEfeEvn.png!mobile)
![_step_false](https://img1.tuicool.com/I7nMv2Z.png!mobile)

- _Setp确实有两个重载函数，如果_Ty是一个数组类型，继续调用_Setpd(_Px,default_delete<_Ux[]>{})。这里根据_Ty的类型我们选择了正确的删除器类型

![_strp_3](https://img0.tuicool.com/eqeQVf6.png!mobile)

- 在_Setpd函数中我们看到了_Ref_count_resource这个类的动态生成。
- 如果_Ty不是一个数组类型呢？我们看一下_Setp的另一个版本，其中出现了_Ref_count这个类的动态生成。
- 两个版本均继续调用了_Set_ptr_rep_and_enable_shared函数，区别就是生成的引用计数类不一样。现在我们搞清楚了大致流程：根据构造函数的不同、模板参数的不同，shared_ptr生成对应的引用计数派生类传入底层函数。_Set_ptr_rep_and_enable_shared这个函数后续做了哪些工作，因为涉及到了weak_ptr和enable_shared_from_this这些类，这里一时半会说不清楚。目前我们只要知道将_Px和_Dt赋值给了基类中那两个成员变量指针即可。
- 到目前为止我们能够明白：不同的shared_ptr对象之所以能够共享资源，是因为其每个对象都有一个指向该资源的指针_Ptr和一个指向引用计数类的指针_Rep，根据删除和内存分配等不同要求引用计数类有多个派生类，在构造shared_ptr时会根据情况生成对应的派生类。
- 我们以一个拷贝构造函数为例，看shared_ptr是如何通过调用其基类提供的函数修改引用计数，达到共享资源的目的的：

![_shared_ptr_copy](https://img2.tuicool.com/qQjiaaz.png!mobile)
![_shared_ptr_copy2](https://img0.tuicool.com/MBNvAbF.png!mobile)

如果_Other._Rep不为空指针，则调用其_Incref()函数，然后对_Ptr和_Rep进行赋值，_Incref()这个函数看名字其实能看出来，就是增加引用计数，前面提到过，这些函数能保证在不同线程中原子的修改引用计数，我们看下内部的实现过程：

![_incref](https://img0.tuicool.com/beYfmen.png!mobile)
![MT_INCR](https://img2.tuicool.com/3aeYzym.png!mobile)

再往下就是和平台实现及其密切的原生API了，我们的分析都到这一步为止。
我们再看一下shared_ptr的析构函数

![~shared_ptr](https://img1.tuicool.com/ZVNvmaR.png!mobile)
![_Decref](https://img2.tuicool.com/6RBFbaq.png!mobile)
![_Decref](https://img1.tuicool.com/iEVbuye.png!mobile)
![_DecWref](https://img1.tuicool.com/YnyyeeE.png!mobile)

- 析构函数中，如果_Uses减1之后等于0，则调用_Destroy()函数释放资源，并调用_Decwref()函数对弱引用计数减1，由于弱引用的存在，哪怕资源已经释放，也有可能有弱引用绑定到引用计数类上，所以这里不能直接释放引用计数类的资源，而是判断弱引用计数是否为0，如果_Weaks也为0，那么可以释放引用计数类的资源，调用_Delete_this()。