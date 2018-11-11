## C++ STL  Allocator

接下来，我们要进入STL我认为最复杂的东西了，Allocator，即空间适配器，那么allocator明明应该是内存啊，为啥是空间呢，因为通过自己定义的Allocator你可以向任何区域索取空间，内存、硬盘都可以，所以空间似乎更符合定义。

### Why Allocator

我们需要一个灵活的方式去，创建对象，销毁对象，分配内存。这就是我们需要allocator的原因，但是为什么把allocator作为一个模版参数呢？allocator作为一个分配内存的工具，却定义在template中显得有些不合逻辑。这是因为在早期的C++，STLer苦心孤诣想避免virtual function这个”妖魔鬼怪“。

```c++
template<typename T, typename Alloc>
Class Vec{
	Vec(){}
}
template<typename T>
Class Vec{
    Vec(BaseAlocator* alloc){
        m_alloc = alloc;
    }
    BaseAlocator* m_alloc;
}
```

我们比较上面两种定义allocator的方式，不难发现，如果我们通过Base Class来管理allocator的话，免不了调用virtual function。调用函数就必须在runtime时才能彻底知道,which is dynamic dispath。而通过template我们避免了dynamic dispath而采用static dispatch。C++追求的是**zero cost abstraction**, 在这个有些信条化的概念指导下，Allocator作为一个模版参数就这样被引入了。

具体到功能上看，allocator除了负责内存释放分配，对象的contructor和deconstructor。

### Basic Allocator

下面是STL定义的allocator需要提供的interface:

```cpp
// 以下几种自定义类型是一种type_traits技巧，暂时不需要了解
allocator::value_type
allocator::pointer
allocator::const_pointer
allocator::reference
allocator::const_reference
allocator::size_type
allocator::difference

// 一个嵌套的(nested)class template，class rebind<U>拥有唯一成员other，那是一个typedef，代表allocator<U>
allocator::rebind

allocator::allocator() // 默认构造函数
allocator::allocator(const allocator&) // 拷贝构造函数
template <class U>allocator::allocator(const allocator<U>&) // 泛化的拷贝构造函数
allocator::~allocator() // 析构函数

// 返回某个对象的地址，a.address(x)等同于&x
pointer allocator::address(reference x) const
// 返回某个const对象的地址，a.address(x)等同于&x
const_pointer allocator::address(const_reference x) const
// 配置空间，足以存储n个T对象。第二个参数是个提示。实现上可能会利用它来增进区域性(locality)，或完全忽略之
pointer allocator::allocate(size_type n, const void* = 0)
// 释放先前配置的空间
void allocator::deallocate(pointer p, size_type n)
// 返回可成功配置的最大量
size_type allocator:maxsize() const
// 调用对象的构造函数，等同于 new((void*)p) T(x)
void allocator::construct(pointer p, const T& x)
// 调用对象的析构函数，等同于 p->~T()
void allocator::destroy(pointer p)
```

根据这个STL提供的allocator定义，我们可以写出自己定义的allocator实现。

```cpp
template <class T>
inline T* _allocate(ptrdiff_t size, T*) {
	set_new_handler(0); //设置malloc失败的处理函数，这里例子为0即不处理
	T* tmp = (T*)(::operator new((size_t)(size * sizeof(T))));
	if (tmp == 0) {
		cerr << "out of memory" << endl;
		exit(1);
	}
	return tmp;
}
template <class T>
inline void _deallocate(T* buffer) {
	::operator delete(buffer);
}
template <class T1, class T2>
inline void _construct(T1* p, const T2& value) {
	new(p) T1(value); // placement new. invoke ctor of T1.
}
template <class T>
inline void _destroy(T* ptr) {
	ptr->~T();
}
template <class T>
class allocator {
	public:
    typedef Tvalue_type;
    typedef T* pointer;
    typedef const T* const_pointer;
    typedef T& reference;
    typedef const T& const_reference;
    typedef size_t size_type;
    typedef ptrdiff_tdifference_type;
    // rebind allocator of type U
	template <class U>
    struct rebind {
		typedef allocator<U> other;
	};
// hint used for locality. ref.[Austern],p189
    pointerallocate(size_type n, const void* hint=0) {
        return _allocate((difference_type)n, (pointer)0);
    }
    void deallocate(pointer p, size_type n) { _deallocate(p); }
    void construct(pointer p, const T& value) {
        _construct(p, value);
    }
    void destroy(pointer p) {_destroy(p); }
    pointeraddress(reference x) { return (pointer)&x; }
    const_pointerconst_address(const_reference x) {
        return (const_pointer)&x;
    }
    size_typemax_size() const {
    return size_type(UINT_MAX/sizeof(T));
    }
};
```

然而，SGI STL与标准规范不同，其名称为alloc而不是allocator，也就是说如果你想在程序中采用SGI STL的allocator你需要这么写：`` vector<int,std::alloc> iv;``
在这点上，我们不必为这种不一致而困扰，因为通常我们采用default的allocator，而SGI STL的每一个container都制定了其default的allocator为alloc。

### SGI STL alloc

其实，SGI STL标准为我们已经提供了一个符合STL标准的allocator，但是这个allocator只是对`new delete`自做了一个简单的封装而不常用。

在SGI STL内部，有一个更robust的allocator实现，它完全摒弃了STL allocator的标准，下面我们来看它的实现。

```cpp
class Foo{}
Foo* p = new Foo();
delete p;
```

我们观察上面简单的C++一段很常用的逻辑，在这段逻辑中，new的行为有两个:

1. 调用`::operator new`分配内存.
2. 调用default Constuctor of Foo.

同理，delete也分为两步：

1. 调用`~Foo()`进行析构。
2. `::operator delete`释放内存。

不难发现上述的功能也可以分为两大块，一是对内存的管理，二是对对象的管理。这两个功能分别对应`#include<stl_construct.h>`的`construct() destroy()`和`#include<stl_alloc.h>`的`alloc`.

| `stl_construct.h`     | 定义construct和destroy，负责对象的建构和析构，隶属于STL      |
| --------------------- | ------------------------------------------------------------ |
| `stl_alloc.h`         | 定义一二级配置器，彼此合作，配置器名为alloc                  |
| `stl_uninitialized.h` | 定义一些全全局函数表达式，用来填充复制大块内存区间。`un_initialized_copy()` `un_initialized_fill()` `un_initialized_filln()` |

