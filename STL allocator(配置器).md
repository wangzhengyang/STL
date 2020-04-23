# STL allocator(配置器)

## 头文件

```c++
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>
#include <string.h>
#include <assert.h>
```

## STL 第一级配置器

在第一级配置器当中，内存的分配以及释放只是通过`malloc` `realloc` `free`来实现

### 类定义

```c++
//类定义
template<int __inst>
class __malloc_alloc_template{
private:
    static void *_S_oom_malloc(size_t); //malloc失败后，调用的处理函数
    static void *_S_oom_realloc(void*, size_t); //realloc失败后，调用的处理函数
    
    static void (*__malloc_alloc_oom_handler)(); //函数指针，这里设置出错处理函数
    
public:
    static void *allocate(size_t __n);  //内存分配
    static void *deallocate(void *__p, size_t ); //内存释放
    static void *reallocate(void *__p, size_t , size_t __new_sz); //内存在申请
    static void (*__set_malloc_handler(void (*__f)()))(); //设置出错函数
};
```

### 初始化

```c++
template<int __inst>
void (* __malloc_alloc_template::__malloc_alloc_oom_handler)() = 0; //出错函数初始化

typedef __malloc_alloc_template<0> malloc_alloc;	//别名定义
```

### 函数定义

#### 内存分配

```c++
template <int __inst>
void *__malloc_alloc_template<__inst>::allocate(size_t __n)
{
    void *__result = malloc(__n); //直接调用malloc
    if(0 == __result){ //申请失败
        __result = _S_oom_malloc(__n); //再次申请，并且调用出错处理函数，在该函数中存在死循环
    }
    return __result;
}
```

#### 内存释放

```c++
template <int __inst>
void *__malloc_alloc_template<__inst>::deallocate(void *__p, size_t __n) //参数__n不使用
{
    free(__p); //直接调用free
}
```

#### 内存再申请

```c++
template <int __inst>
void *__malloc_alloc_template<__isnt>::reallocate(void* __p, size_t __old_sz, size_t __new_sz) //__old_sz未使用
{
    void *__result = realloc(__p, __new_sz); //直接调用realloc
    if(0 == __result){
        __result = _S_oom_realloc(__p, __new_sz); //失败，再次申请
    }
    return __result;
}
```

#### 设置出错

```c++
template <int __inst>
void (* __malloc_alloc_template<__inst>::__set_malloc_handler(void (*__f)()))()
{
    void (*__old)() = __malloc_alloc_oom_handler;
    __malloc_alloc_oom_handler = __f;
    return (__old);
}
```

#### `malloc`失败再申请

```c++
template <int __inst>
void *__malloc_alloc_template<__inst>::_S_oom_malloc(size_t __n)
{
    void (*__my_alloc_handler)();
    void *__result;
    
    for(;;){ //死循环，当申请到内存时退出
        __my_alloc_handler = __malloc_alloc_oom_handler;
        if(0 == __my_malloc_handler){/*TODO*/}
        (*__mu_alloc_handler)(); //出错处理函数
        __result = malloc(__n);
        if(__result) return (__result);
    }
}
```

#### realloc失败再申请

```c++
template <int __inst>
void *__malloc_alloc_template<__inst>::_S_oom_realloc(void *__p, size_t __n)
{
    void (*__my_alloc_handler)();
    void *__result;
    
    for(;;){ //死循环，当申请到内存时退出
        __my_alloc_handler = __malloc_alloc_oom_handler;
        if(0 == __my_malloc_handler){/*TODO*/}
        (*__mu_alloc_handler)(); //出错处理函数
        __result = realloc(__n);
        if(__result) return (__result);
    }    
}
```

## STL 第二级配置器

#### 类定义

```c++
template <bool __threads, int __inst> //__threads由于维护全局变量，所以多线程在申请内存时需要互斥处理
class __default_alloc_template{
private:
    enum{_ALIGN = 8}; //所有申请的内存大小都将转化为8的整数倍
    enum{_MAX_BYTES = 128}; //当申请内存大于128字节时，使用第一级配置器申请，否则使用第二级配置器
    enum{_NFREELISTS = 16}; //128/8=16 定义一个数组链表，从0到15每个链表维护着8的整数倍的内存
    
    union _Obj{ //大小为指针大小 4bytes
        union _Obj* _M_free_list_link; //
        char _M_client_data[1];
    }
    
    static _Obj *volatile _S_free_list[_NFREELISTS]; //定义一个16个空闲链表数组，初始化为0
    
    static char *_S_start_free; //内存池起始位置
    static char *_S_end_free;	//内存池结束位置
    static size_t _S_heap_size;	//内存池大小
    
    static size_t _S_round_up(size_t __bytes); //将申请字节个数转化为8的倍数
    static size_t _S_freelist_index(size_t __bytes); //通过字节数快速定位需要从哪个链表中申请内存，这里有个小技巧，由于大于128bytes都通过第一级配置器申请了，所以该函数最大的返回索引只能是15，并不会超出_S_free_list的数组长度
    static void *_S_refill(size_t __n); 
    static char *_S_chunk_alloc(size_t __size, int &__nobjs); 

public:
    static void *allocate(size_t __n); //申请内存
    static void *deallocate(void *__p, size_t __n); //释放内存
    static void *reallocate(void *__p, size_t __old_sz, size_t __new_sz);//重新申请内存
};
```

#### 初始化

```c++
template <bool __threads, int __inst>
char *__default_alloc_template<__threads, __inst>::_S_start_free = 0;

template <bool __threads, int __inst>
char *__default_alloc_template<__threads, __inst>::_S_end_free = 0;

template <bool __threads, int __inst>
size_t __default_alloc_template<__threads, __inst>::_S_heap_size = 0;

template <bool __threads, int __inst>
typename __default_alloc_template<__threads, __inst>::Obj *volatile __default_alloc_template<__threads, __inst>::_S_free_list[__default_alloc_template<__threads, __inst>::_NFREELISTS] = { 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0}; //初始化为空，表明没有空闲的链表可供使用

typedef __default_alloc_template<__NODE_ALLOCATOR_THREADS, 0> alloc; //别名
typedef __default_alloc_template<false, 0> signle_client_alloc; //别名
```

### 函数定义

#### 8的整数倍字节数

```c++
template <bool __threads, int __inst>
size_t __default_alloc_template::_S_round_up(size_t __bytes)
{
    return (((__bytes) + (size_t)_ALIGN - 1) &~((size_t)_ALIGN - 1));
}
```

#### 获取空闲链表索引

```c++
template <bool __threads, int __inst>
size_t __default_alloc_template::_S_freelist_index(size_t __bytes)
{
    return (((__bytes) + (size_t)_ALIGN - 1)/(size_t)_ALIGN - 1);
}
```

#### 内存分配

```c++
template <bool __threads, int __inst>
void *__default_alloc_template::allocate(size_t __n)
{
    void *__ret = 0;
    if(__n > (size_t)_MAX_BYTES){ //大于128个字节
        __ret = malloc_alloc::allocate(__n); //直接调用第一级配置器
    }else{
        _Obj* volatile* __my_free_list = _S_free_list + _S_freelist_index(__n); //获取索引
        _Obj* __result = *__my_free_list; //获取信息
        if(__result == 0){ //如果没有空闲的内存
            __ret = _S_refill(_S_round_up(__n)); //申请内存
        }else{
            *__my_free_list = __result->_M_free_list_link; //指向下一个空闲的内存
            __ret = __result;
        }
    }
    return __ret;
}
```

#### 内存释放

```c++
template <bool __threads, int __inst>
void *__default_alloc_template::deallocate(void *__p, size_t __n)
{
    if(__n > (size_t)_MAX_BYTES){//大于128个字节
        malloc_alloc::deallocate(__p, __n); //直接调用第一级配置器
    }else{
        _Obj* volatile* __my_free_list = _S_free_list + _S_freelist_index(__n);
        _Obj* __q = (_Obj*)__p;
        __q->_M_free_list_link = *__my_free_list;
        *__my_free_list = __q;
    }
}
```

#### 内存再申请

```c++
template <bool __threads, int __inst>
void *_default_alloc_template<__threads, __inst>::reallocate(void *__p, size_t __old_sz, size_t __new_sz)
{
    void *__result;
    size_t __copy_sz;
    if(__old_sz > (size_t)_MAX_BYTES && __new_sz > (size_t)_MAX_BYTES){
        return (realloc(__p, __new_sz));
    }
    if(_S_round_up(__old_sz) == _S_round_up(__new_sz)) return (__p);
    __result = allocate(__new_sz);
    __copy_sz = __new_sz > __old_sz ? __old_sz : __new_sz;
    memcpy(__result, __p, __copy_sz);
    deallocate(__p, __old_sz);
    return (__result);
}
```

#### 内存调用_S_chunk_alloc申请内存

```c++
template <bool __threads, int __inst>
void *__default_alloc_template<__threads, __inst>::_S_refill(size_t __n)
{
    int __nobjs = 20; 
    char *__chunk = _S_chunk_alloc(__n, __nobjs); //申请20*_n的内存块 但返回的结果有可能不是20块
    _Obj* volatile* __my_free_list;
    _Obj* __result;
    _Obj* __current_obj;
    _Obj* __next_obj;
    int __i;
    
    if(1 == __nobjs) return (__chunk); //只申请到一块，直接返回给用户，因为只需要一块
    __my_free_list = _S_free_list + _S_freelist_index(__n); //查找索引
    __result = (_Obj*)__chunk; //返回第一个块地址
    *__my_free_list = __next_obj = (_Obj*)(__chunk + _n);//记录第一个块后的第二个块起始位置
    for(__i = 1; ; __i++){//将剩余的块通过链表连接起来
        __current_obj = __next_obj;
        __next_obj = (_Obj*)((char*)__next_obj + __n);
        if(__nobjs - 1 == __i){
            __current_obj->_M_free_list_link = 0;
            break;
        }else{
            __current_obj->_M_free_list_link = __next_obj;
        }
    }
    return (__result);
}
```

#### 内存堆申请

```c++
template <bool __threads, int __inst>
char *__default_alloc_template<__threads, __inst>::_S_chunk_alloc(size_t __size, int &__nobjs){
    char *__result;
    size_t __total_bytes = __size * __nobjs; //申请的内存大小
    size_t __bytes_left = _S_end_free - _S_start_free; //剩余内存大小
    if(__bytes_left >= __total_bytes){ //申请的内存大于剩余内存
        __result = _S_start_free;
        return (__result);
    }else if(__bytes_left >= __size){ //剩余的内存大于一个块
        __nobjs = (int)(__bytes_left/__size); //计算到底可以申请多少个块
        __total_bytes = __size * __nobjs; //可以申请多少字节
        __result = _S_start_free;
        _S_start_free += __total_bytes; //堆的起始位置修改
        return (__result);
    }else{ //一块内存都没有
        size_t __bytes_to_get = 2 * __total_bytes + _S_round_up(_S_heap_size >> 4); //计算申请内存大小
        if(__bytes_left > 0){ //还有剩余内存
            _Obj* volatile* __my_free_list = _S_free_list + _S_freelist_index(__bytes_left); //获取索引
            ((_Obj*)_S_start_free)->_M_free_list_link = *__my_free_list; //把该内存放入其他链表中
            *__my_free_list = (_Obj*)_S_start_free;
        }
        _S_start_free = (char*)malloc(__bytes_to_get); //申请内存
        if(0 == _S_start_free){ //没有申请到内存
            size_t __i;
            _Obj* volatile* __my_free_list;
            _Obj* __p;
            for(__i == __size; __i <= (size_t)_MAX_BYTES; __i +=(size_t)_ALIGN){ //去后面的链表查找，看有没有空闲的块没有使用
                __my_free_list = _S_free_list + _S_freelist_index(__i);
                __p = *__my_free_list;
                if(0 != __p){//有空闲的块
                    *__my_free_list = __p->_M_free_list_link;
                    _S_start_free = (char*)__p;//以第一个块重新设置堆
                    _S_end_free = _S_start_free + i;
                    return (_S_chunk_alloc(__size, __nobjs));//递归重新去获取
                }
            }
            //没有空闲的块
            _S_end_free = 0;
            _S_start_free = (char*)malloc_alloc::allocate(__bytes_to_get);//调用第一级配置器(里面有一个死循环，可以反复尝试申请内存)
        }
        //申请到内存 修改堆属性
        _S_heap_size += __bytes_to_get;
        _S_end_free = _S_start_free * __bytes_to_get;
        //到这里 只是申请堆大小但是并没有给对象分配内存，所以递归一次调用自己，这样就可以申请到对象内存
        return (_S_chunk_alloc(__size, __nobjs));
    }
}
```

