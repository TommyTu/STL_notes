## vector

弃坑了一年，为了准备HRT面试，又来补坑了，这家观察电面只考list和vector，于是懒癌发作我就只写写vector和list吧。

### vector 介绍
vector其实和array很像，都是本质上都是连续空间的数组，但是无论是从heap还是stack方式分配的array都是fixed size的，但是很多时候，我们并不知道我们需要
多少空间，这就是我们为什么要一个灵活的容器。vector便闪耀登场，对于一个dynamic size的array来讲，我们不免担心，怎么动态的改变大小以及释放空间是最优的
策略呢？如果做到这两点，便是vector的关键。


### vector的迭代器
```
template <class T, classAlloc = alloc>
class vector {
public:
typedef T value_type;
typedef value_type*iterator;
...
};
```
作为一个连续的数组，vector也理所当然的要支持random access，以及iterator的其他操作，我们可以看到，对于vector来讲iterator其实就是T的pointer。

### vector的数据结构
```
template <class T, classAlloc = alloc>
class vector {
...
protected:
iterator start;
iterator finish;
iterator end_of_storage; 
...
};
```
这便是vector的核心，除了标准的start 和 finish，标志着连续数组的头和尾，还有个end_of_storage，end_of_storage可以理解成vector目前的容量，当我们
继续不断的`push_back`的时候，如果finish没有超过end_of_storage，事情尤其简单，只需`finish++`并且填充空间就行了，一旦finish和end_of_storage相
等时，便是一个warning我们已经用完了capacity了，需要再找一块地方了。同时通过这三个iterator，可以完成很多很基本的操作：
```
template <class T, classAlloc = alloc>
class vector {
...
public:
iterator begin() { return start; }
iterator end() { return finish; }
size_type size() const { return size_type(end() - begin()); }
size_type capacity() const {
return size_type(end_of_storage - begin()); }
bool empty() const { return begin() == end(); }
T& operator[](size_type n) { return*(begin() + n); }
T& front() { return *begin(); }
```

### vector的核心操作：constructor and push_back
现在，问题来了，如果我们作为一个用户使用vector，并使用我们最常见的操作：若干个push_back，这中间会发生什么呢？
首先，vector by default使用alloc作为空间适配器，并且定义了一个data_allocator:
```
template <class T, classAlloc = alloc>
class vector {
protected:
  typedef simple_alloc<value_type,Alloc> data_allocator;
...
};
```
对于constructor来说，其实就是创建一个空间，并填充：
```
vector(size_type n, const T& value) {fill_initialize(n, value); }
// 填充并初始化
void fill_initialize(size_type n, const T& value) {
start =allocate_and_fill(n, value);
finish = start + n;
end_of_storage = finish;
}

iterator allocate_and_fill(size_type n, const T& x) {
  iterator result =data_allocator::allocate(n); 
  uninitialized_fill_n(result, n, x); 
  return result;
}
```
当我们push_back的时候，我们便进行判断，如果空间超过了capacity，便扩张，否则，只填充：
```
void push_back(const T& x) {
if (finish != end_of_storage) { 
  construct(finish, x);
  ++finish;
}
else {
  insert_aux(end(), x);
}

template <class T, class Alloc>
void vector<T, Alloc>::insert_aux(iterator position, const T& x) {
if (finish != end_of_storage) {
  // 空间没用完时，我们在后面新建一个以当前finish位默认值的元素。
  construct(finish, *(finish - 1));
  ++finish;
  T x_copy = x;
  // 将被插入的地方到finish-2的地方，复制到终于finish-1的地方，[href=https://zh.cppreference.com/w/cpp/algorithm/copy_backward]参考
  copy_backward(position, finish - 2, finish - 1);
  *position = x_copy;
} else { 
  const size_type old_size = size();
  //我们可以看见capacity的扩张是每次翻一倍。
  const size_typelen = old_size != 0 ? 2 * old_size : 1;
  iterator new_start =data_allocator::allocate(len); 
  iterator new_finish = new_start;
  //把之前的复制到新的vector，插入新元素，然后把position右侧的也copy到vector
  try {
    new_finish = uninitialized_copy(start, position, new_start);
    construct(new_finish, x);
    ++new_finish;
    new_finish =uninitialized_copy(position, finish, new_finish);
  }
  catch(...) {
    // "commit or rollback" semantics.
    destroy(new_start, new_finish);
    data_allocator::deallocate(new_start, len);
    throw;
  }
  //解构原来的vector
  destroy(begin(), end());
  deallocate();
  //调整迭代器
  start = new_start;
  finish = new_finish;
  end_of_storage = new_start + len;
}
```
所以其实本质来讲，vector的扩张会开辟一个新的vector并把之前的vector复制过来，所以之前保存的iterator会都失效了。









