#c-cpp/stl-code #book 
## 2022.6.23

## p116

Vector is implemented as a base class `_Vector_base` and `vector` itself. As the book said, both of them are in `stl_vector.h`.

`.../bits/stl_vector.h[422]`
```cpp
  template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
    class vector : protected _Vector_base<_Tp, _Alloc>
    {
        ...
```

`.../bits/stl_vector.h[84]`
```cpp
  template<typename _Tp, typename _Alloc>
    struct _Vector_base
    {
        ...
```

The definition of the allocator has changed.

`.../bits/stl_vector.h[87]`
```cpp
      typedef typename __gnu_cxx::__alloc_traits<_Alloc>::template
	rebind<_Tp>::other _Tp_alloc_type; // template<typename _Tp, typename _Alloc> struct _Vector_base
      typedef typename __gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer
       	pointer;
```

It uses `rebind` to change the type parameter of the allocator to the template parameter which the vector received. Such as if we define `std::vector<int, __gnu_cxx::malloc_allocator<double>>`, then the vector will rebind the inner allocator to parameter of `int`.

Besides, this `rebind` is not just the `rebind` structure which defined in allocators, because some allocators don't implement the `rebind` method.

`.../ext/alloc_traits.h[118]`
```cpp
    template<typename _Tp>
      struct rebind // __alloc_traits::rebind
      { typedef typename _Base_type::template rebind_alloc<_Tp> other; };
```

`.../ext/alloc_traits.h[55]`
```cpp
    typedef std::allocator_traits<_Alloc>           _Base_type; // __alloc_traits::_Base_type
```

`.../bits/alloc_traits.h[212]`
```cpp
      template<typename _Tp>
	using rebind_alloc = __alloc_rebind<_Alloc, _Tp>; // allocator_traits::rebind_alloc
```

`.../bits/alloc_traits.h[78]`
```cpp
  template<typename _Alloc, typename _Up>
    using __alloc_rebind // __allocator_traits_base::__alloc_rebind
      = typename __allocator_traits_base::template __rebind<_Alloc, _Up>::type;
```

`.../bits/alloc_traits.h[51]`
```cpp
    template<typename _Tp, typename _Up, typename = void>
      struct __rebind : __replace_first_arg<_Tp, _Up> { };

    template<typename _Tp, typename _Up>
      struct __rebind<_Tp, _Up,
		      __void_t<typename _Tp::template rebind<_Up>::other>>
      { using type = typename _Tp::template rebind<_Up>::other; };
```

As a result, if the allocator has implemented the `rebind` structure, we will use it, otherwise, we use `__replace_first_arg` to construct a new allocator by replacing the argument in `allocator<_Tp>`.

`.../bits/ptr_traits.h[65]`
```cpp
  template<typename _Tp, typename _Up>
    struct __replace_first_arg
    { };

  template<template<typename, typename...> class _SomeTemplate, typename _Up,
           typename _Tp, typename... _Types>
    struct __replace_first_arg<_SomeTemplate<_Tp, _Types...>, _Up>
    { using type = _SomeTemplate<_Up, _Types...>; };
```

For `std::vector<int, __gnu_cxx::malloc_allocator<double>>`, we will finally call `__replace_first_arg<__gnu_cxx::malloc_allocator<double>, int>`, then get the `type` `__gnu_cxx::malloc_allocator<int>` as `_Tp_alloc_type`.

## p116

The implementation of `vector` is much more complicated than old version. The three attribute `iterator` have been moved to `struct _Vector_base::_Vector_impl_data`.

`.../bits/stl_vector.h[92]`

```cpp
      struct _Vector_impl_data
      {
	pointer _M_start;
	pointer _M_finish;
	pointer _M_end_of_storage;
```

And in `class vector`, we are using the `struct _Vector_impl` which inheriting from it.

`.../bits/stl_vector.h[133]`

```cpp
      struct _Vector_impl
	: public _Tp_alloc_type, public _Vector_impl_data
```

Meanwhile, some methods in vector are defined in `vector.tcc`, such as `emplace_back`

`.../bits/vector.tcc[102]`

```cpp
#if __cplusplus >= 201103L
  template<typename _Tp, typename _Alloc>
    template<typename... _Args>
#if __cplusplus > 201402L
      _GLIBCXX20_CONSTEXPR
      typename vector<_Tp, _Alloc>::reference
#else
      void
#endif
      vector<_Tp, _Alloc>::
      emplace_back(_Args&&... __args)
      {
	if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage)
	  {
	    _GLIBCXX_ASAN_ANNOTATE_GROW(1);
	    _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
				     std::forward<_Args>(__args)...);
	    ++this->_M_impl._M_finish;
	    _GLIBCXX_ASAN_ANNOTATE_GREW(1);
	  }
	else
	  _M_realloc_insert(end(), std::forward<_Args>(__args)...);
#if __cplusplus > 201402L
	return back();
#endif
      }
#endif
```

## p117

The iterator of the `vector`:

`.../bits/stl_vector.h[453]`
```cpp
      typedef __gnu_cxx::__normal_iterator<pointer, vector> iterator;
      typedef __gnu_cxx::__normal_iterator<const_pointer, vector>
      const_iterator;
      typedef std::reverse_iterator<const_iterator>	const_reverse_iterator;
      typedef std::reverse_iterator<iterator>		reverse_iterator;
```

`.../bits/stl_iterator.h[1042]`

```cpp
  template<typename _Iterator, typename _Container>
    class __normal_iterator
    {
    protected:
      _Iterator _M_current;

      typedef std::iterator_traits<_Iterator>		__traits_type;
```

Meanwhile, we notice that `gcc` is now using `pointer` instead of `iterator`, so we should pay attention to the definition of `pointer`.

`.../bits/stl_vector.h[92]`

```cpp
      struct _Vector_impl_data
      {
	pointer _M_start;
	pointer _M_finish;
	pointer _M_end_of_storage;
```

`.../bits/stl_vector.h[92]`

```cpp
      typedef typename __gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer pointer; // _Vector_base::pointer
```

`.../ext/alloc_traits.h[55]`

```cpp
    typedef std::allocator_traits<_Alloc>           _Base_type;
    typedef typename _Base_type::value_type         value_type;
    typedef typename _Base_type::pointer            pointer; // __gnu_cxx::__alloc_traits<_Alloc>::pointer
```

`.../ext/alloc_traits.h[95]`

```cpp
      typedef typename _Alloc::value_type value_type;
      using pointer = __detected_or_t<value_type*, __pointer, _Alloc>; // std::allocator_traits<_Alloc>::pointer
```


`.../ext/alloc_traits.h[60]`
```cpp
    template<typename _Tp>
      using __pointer = typename _Tp::pointer; // __allocator_traits_base::__pointer
```

So the `pointer` in `vector` is `Alloc::pointer` if that type exists, otherwise `value_type*`.

We take `__new_allocator` as an example because it is the default `Alloc` of `vector` in `gcc12.1`. Before `c++17`, there still existed `Alloc::pointer` which is `_Tp*`, however, after `c++17`, there only exist `value_type`. No matter how, `pointer` is an alias of `_Tp*`.

`.../bits/new_allocator.h[55]`
```cpp
  template<typename _Tp>
    class __new_allocator
    {
    public:
      typedef _Tp        value_type;
      typedef std::size_t     size_type;
      typedef std::ptrdiff_t  difference_type;
#if __cplusplus <= 201703L
      typedef _Tp*       pointer;
      typedef const _Tp* const_pointer;
      typedef _Tp&       reference;
      typedef const _Tp& const_reference;
```

## p118

Take `begin` as an example:

`.../bits/stl_vector.h[866]`

```cpp
      _GLIBCXX_NODISCARD _GLIBCXX20_CONSTEXPR
      iterator
      begin() _GLIBCXX_NOEXCEPT
      { return iterator(this->_M_impl._M_start); }
```

As you can see, `begin` doesn't return a `pointer` directly, but wraps it as an `iterator`.

## p121

About the constructor of `vector`:

`.../bits/stl_vector.h[563]`
```cpp
      _GLIBCXX20_CONSTEXPR
      vector(size_type __n, const value_type& __value,
	     const allocator_type& __a = allocator_type())
      : _Base(_S_check_init_len(__n, __a), __a)
      { _M_fill_initialize(__n, __value); }
```

The initialization is finished by `_Base` using delegate constructor.

`.../bits/stl_vector.h[329]`
```cpp
      _GLIBCXX20_CONSTEXPR
      _Vector_base(size_t __n, const allocator_type& __a)
      : _M_impl(__a)
      { _M_create_storage(__n); }
```

`.../bits/stl_vector.h[391]`
```cpp
      _GLIBCXX20_CONSTEXPR
      void
      _M_create_storage(size_t __n)
      {
	this->_M_impl._M_start = this->_M_allocate(__n);
	this->_M_impl._M_finish = this->_M_impl._M_start;
	this->_M_impl._M_end_of_storage = this->_M_impl._M_start + __n;
      }
```

`.../bits/stl_vector.h[373]`
```cpp
      _GLIBCXX20_CONSTEXPR
      pointer
      _M_allocate(size_t __n)
      {
	typedef __gnu_cxx::__alloc_traits<_Tp_alloc_type> _Tr;
	return __n != 0 ? _Tr::allocate(_M_impl, __n) : pointer();
      }
```

As to the body of the function:

`.../bits/stl_vector.h[1697]`
```cpp
      _GLIBCXX20_CONSTEXPR
      void
      _M_fill_initialize(size_type __n, const value_type& __value)
      {
	this->_M_impl._M_finish =
	  std::__uninitialized_fill_n_a(this->_M_impl._M_start, __n, __value,
					_M_get_Tp_allocator());
      }
```

`.../bits/stl_vector.h[296]`
```cpp
      _GLIBCXX20_CONSTEXPR
      _Tp_alloc_type&
      _M_get_Tp_allocator() _GLIBCXX_NOEXCEPT
      { return this->_M_impl; }
```

`_M_impl` is of class `_Vector_impl`, which inherits from `_Tp_alloc_type`, so it can be seen as an allocator.

`_Tp_alloc_type` is the allocator class allocates the space of vector's type.

```cpp
      struct _Vector_impl
	: public _Tp_alloc_type, public _Vector_impl_data
```

## p122

`push_back` has two implementation, the first version will be called if passing an `lvalue-reference`, and the second version will be called if passing an `rvalue-reference`.

`.../bits/stl_vector.h[1274]`
```cpp
      _GLIBCXX20_CONSTEXPR
      void
      push_back(const value_type& __x)
      {
	if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage)
	  {
	    _GLIBCXX_ASAN_ANNOTATE_GROW(1);
	    _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
				     __x);
	    ++this->_M_impl._M_finish;
	    _GLIBCXX_ASAN_ANNOTATE_GREW(1);
	  }
	else
	  _M_realloc_insert(end(), __x);
      }

#if __cplusplus >= 201103L
      _GLIBCXX20_CONSTEXPR
      void
      push_back(value_type&& __x)
      { emplace_back(std::move(__x)); }
```

`.../bits/vector.tcc[434]`
```cpp
#if __cplusplus >= 201103L
  template<typename _Tp, typename _Alloc>
    template<typename... _Args>
      _GLIBCXX20_CONSTEXPR
      void
      vector<_Tp, _Alloc>::
      _M_realloc_insert(iterator __position, _Args&&... __args)
#else
  template<typename _Tp, typename _Alloc>
    void
    vector<_Tp, _Alloc>::
    _M_realloc_insert(iterator __position, const _Tp& __x)
#endif
    {
      const size_type __len =
	_M_check_len(size_type(1), "vector::_M_realloc_insert");
      pointer __old_start = this->_M_impl._M_start;
      pointer __old_finish = this->_M_impl._M_finish;
      const size_type __elems_before = __position - begin();
      pointer __new_start(this->_M_allocate(__len));
      pointer __new_finish(__new_start);
      __try
	{
	  // The order of the three operations is dictated by the C++11
	  // case, where the moves could alter a new element belonging
	  // to the existing vector.  This is an issue only for callers
	  // taking the element by lvalue ref (see last bullet of C++11
	  // [res.on.arguments]).
	  _Alloc_traits::construct(this->_M_impl,
				   __new_start + __elems_before,
#if __cplusplus >= 201103L
				   std::forward<_Args>(__args)...);
#else
				   __x);
#endif
	  __new_finish = pointer();

#if __cplusplus >= 201103L
	  if _GLIBCXX17_CONSTEXPR (_S_use_relocate())
	    {
	      __new_finish = _S_relocate(__old_start, __position.base(),
					 __new_start, _M_get_Tp_allocator());

	      ++__new_finish;

	      __new_finish = _S_relocate(__position.base(), __old_finish,
					 __new_finish, _M_get_Tp_allocator());
	    }
	  else
#endif
	    {
	      __new_finish
		= std::__uninitialized_move_if_noexcept_a
		(__old_start, __position.base(),
		 __new_start, _M_get_Tp_allocator());

	      ++__new_finish;

	      __new_finish
		= std::__uninitialized_move_if_noexcept_a
		(__position.base(), __old_finish,
		 __new_finish, _M_get_Tp_allocator());
	    }
	}
      __catch(...)
	{
	  if (!__new_finish)
	    _Alloc_traits::destroy(this->_M_impl,
				   __new_start + __elems_before);
	  else
	    std::_Destroy(__new_start, __new_finish, _M_get_Tp_allocator());
	  _M_deallocate(__new_start, __len);
	  __throw_exception_again;
	}
#if __cplusplus >= 201103L
      if _GLIBCXX17_CONSTEXPR (!_S_use_relocate())
#endif
	std::_Destroy(__old_start, __old_finish, _M_get_Tp_allocator());
      _GLIBCXX_ASAN_ANNOTATE_REINIT;
      _M_deallocate(__old_start,
		    this->_M_impl._M_end_of_storage - __old_start);
      this->_M_impl._M_start = __new_start;
      this->_M_impl._M_finish = __new_finish;
      this->_M_impl._M_end_of_storage = __new_start + __len;
    }
```

Take the version after `c++11` as an example:

```cpp
  template<typename _Tp, typename _Alloc>
    template<typename... _Args>
      _GLIBCXX20_CONSTEXPR
      void
      vector<_Tp, _Alloc>::
      _M_realloc_insert(iterator __position, _Args&&... __args)
    {
      const size_type __len =
	_M_check_len(size_type(1), "vector::_M_realloc_insert");
      pointer __old_start = this->_M_impl._M_start;
      pointer __old_finish = this->_M_impl._M_finish;
      const size_type __elems_before = __position - begin();
      pointer __new_start(this->_M_allocate(__len));
      pointer __new_finish(__new_start);
      __try
	{
	  // The order of the three operations is dictated by the C++11
	  // case, where the moves could alter a new element belonging
	  // to the existing vector.  This is an issue only for callers
	  // taking the element by lvalue ref (see last bullet of C++11
	  // [res.on.arguments]).
	  _Alloc_traits::construct(this->_M_impl,
				   __new_start + __elems_before,
				   std::forward<_Args>(__args)...);
	  __new_finish = pointer();

	  if _GLIBCXX17_CONSTEXPR (_S_use_relocate()) // check whether the `_Alloc::value_type` is move-insertable
	    {
	      __new_finish = _S_relocate(__old_start, __position.base(),
					 __new_start, _M_get_Tp_allocator());

	      ++__new_finish;

	      __new_finish = _S_relocate(__position.base(), __old_finish,
					 __new_finish, _M_get_Tp_allocator());
	    }
	  else
	    {
	      __new_finish
		= std::__uninitialized_move_if_noexcept_a
		(__old_start, __position.base(),
		 __new_start, _M_get_Tp_allocator());

	      ++__new_finish;

	      __new_finish
		= std::__uninitialized_move_if_noexcept_a
		(__position.base(), __old_finish,
		 __new_finish, _M_get_Tp_allocator());
	    }
	}
      __catch(...)
	{
	  if (!__new_finish)
	    _Alloc_traits::destroy(this->_M_impl,
				   __new_start + __elems_before);
	  else
	    std::_Destroy(__new_start, __new_finish, _M_get_Tp_allocator()); // commit or rollback
	  _M_deallocate(__new_start, __len);
	  __throw_exception_again;
	}
      if _GLIBCXX17_CONSTEXPR (!_S_use_relocate())
	std::_Destroy(__old_start, __old_finish, _M_get_Tp_allocator());
      _GLIBCXX_ASAN_ANNOTATE_REINIT;
      _M_deallocate(__old_start,
		    this->_M_impl._M_end_of_storage - __old_start);
      this->_M_impl._M_start = __new_start;
      this->_M_impl._M_finish = __new_finish;
      this->_M_impl._M_end_of_storage = __new_start + __len;
    }
```

`.../bits/stl_vector.h[1889]`
```cpp
      _GLIBCXX20_CONSTEXPR
      size_type
      _M_check_len(size_type __n, const char* __s) const
      {
	if (max_size() - size() < __n)
	  __throw_length_error(__N(__s));

	const size_type __len = size() + (std::max)(size(), __n); // if old_len == 0, len = 1, eles len = 2len
	return (__len < size() || __len > max_size()) ? max_size() : __len;
      }
```

After `c++11`, we call `emplace_back` in `push_back`:

`.../bits/vector.tcc[102]`

```cpp
#if __cplusplus >= 201103L
  template<typename _Tp, typename _Alloc>
    template<typename... _Args>
#if __cplusplus > 201402L
      _GLIBCXX20_CONSTEXPR
      typename vector<_Tp, _Alloc>::reference
#else
      void
#endif
      vector<_Tp, _Alloc>::
      emplace_back(_Args&&... __args)
      {
	if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage)
	  {
	    _GLIBCXX_ASAN_ANNOTATE_GROW(1);
	    _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
				     std::forward<_Args>(__args)...);
	    ++this->_M_impl._M_finish;
	    _GLIBCXX_ASAN_ANNOTATE_GREW(1);
	  }
	else
	  _M_realloc_insert(end(), std::forward<_Args>(__args)...);
#if __cplusplus > 201402L
	return back();
#endif
      }
#endif
```

`emplace_back` receives a universal reference but `push_back` receives a const lvalue reference.

## p129

About `__list_node`, two pointers are defined in `_List_node_base`, and the data is defined in `_List_node`:

`.../bits/stl_list.h[233]`
```cpp
  template<typename _Tp>
    struct _List_node : public __detail::_List_node_base
    {
#if __cplusplus >= 201103L
      __gnu_cxx::__aligned_membuf<_Tp> _M_storage;
      _Tp*       _M_valptr()       { return _M_storage._M_ptr(); }
      _Tp const* _M_valptr() const { return _M_storage._M_ptr(); }
#else
      _Tp _M_data;
      _Tp*       _M_valptr()       { return std::__addressof(_M_data); }
      _Tp const* _M_valptr() const { return std::__addressof(_M_data); }
#endif
    };
```

`.../bits/stl_list.h[233]`
```cpp
    struct _List_node_base
    {
      _List_node_base* _M_next;
      _List_node_base* _M_prev;

      static void
      swap(_List_node_base& __x, _List_node_base& __y) _GLIBCXX_USE_NOEXCEPT;

      void
      _M_transfer(_List_node_base* const __first,
		  _List_node_base* const __last) _GLIBCXX_USE_NOEXCEPT;

      void
      _M_reverse() _GLIBCXX_USE_NOEXCEPT;

      void
      _M_hook(_List_node_base* const __position) _GLIBCXX_USE_NOEXCEPT;

      void
      _M_unhook() _GLIBCXX_USE_NOEXCEPT;
    };
```

`.../ext/aligned_buffer.h[46]`
```cpp
  template<typename _Tp>
    struct __aligned_membuf
    {
      // Target macro ADJUST_FIELD_ALIGN can produce different alignment for
      // types when used as class members. __aligned_membuf is intended
      // for use as a class member, so align the buffer as for a class member.
      // Since GCC 8 we could just use alignof(_Tp) instead, but older
      // versions of non-GNU compilers might still need this trick.
      struct _Tp2 { _Tp _M_t; };

      alignas(__alignof__(_Tp2::_M_t)) unsigned char _M_storage[sizeof(_Tp)]; // data is stored here

      __aligned_membuf() = default;

      // Can be used to avoid value-initialization zeroing _M_storage.
      __aligned_membuf(std::nullptr_t) { }

      void*
      _M_addr() noexcept
      { return static_cast<void*>(&_M_storage); }

      const void*
      _M_addr() const noexcept
      { return static_cast<const void*>(&_M_storage); }

      _Tp*
      _M_ptr() noexcept
      { return static_cast<_Tp*>(_M_addr()); }

      const _Tp*
      _M_ptr() const noexcept
      { return static_cast<const _Tp*>(_M_addr()); }
    };
```

## p130

`link_type node;` has been changed to `__detail::_List_node_base* _M_node;` which also point to the real node. If we want to get the data in the node, we should use `static_cast`.

`.../bits/stl_list.h[252]`
```cpp
  template<typename _Tp>
    struct _List_iterator
    {
      typedef _List_iterator<_Tp>		_Self;
      typedef _List_node<_Tp>			_Node;

      typedef ptrdiff_t				difference_type;
      typedef std::bidirectional_iterator_tag	iterator_category;
      typedef _Tp				value_type;
      typedef _Tp*				pointer;
      typedef _Tp&				reference;

      _List_iterator() _GLIBCXX_NOEXCEPT
      : _M_node() { }

      explicit
      _List_iterator(__detail::_List_node_base* __x) _GLIBCXX_NOEXCEPT
      : _M_node(__x) { }

      _Self
      _M_const_cast() const _GLIBCXX_NOEXCEPT
      { return *this; }

      // Must downcast from _List_node_base to _List_node to get to value.
      _GLIBCXX_NODISCARD
      reference
      operator*() const _GLIBCXX_NOEXCEPT
      { return *static_cast<_Node*>(_M_node)->_M_valptr(); }

      _GLIBCXX_NODISCARD
      pointer
      operator->() const _GLIBCXX_NOEXCEPT
      { return static_cast<_Node*>(_M_node)->_M_valptr(); }

      _Self&
      operator++() _GLIBCXX_NOEXCEPT
      {
	_M_node = _M_node->_M_next;
	return *this;
      }

      _Self
      operator++(int) _GLIBCXX_NOEXCEPT
      {
	_Self __tmp = *this;
	_M_node = _M_node->_M_next;
	return __tmp;
      }

      _Self&
      operator--() _GLIBCXX_NOEXCEPT
      {
	_M_node = _M_node->_M_prev;
	return *this;
      }

      _Self
      operator--(int) _GLIBCXX_NOEXCEPT
      {
	_Self __tmp = *this;
	_M_node = _M_node->_M_prev;
	return __tmp;
      }

      _GLIBCXX_NODISCARD
      friend bool
      operator==(const _Self& __x, const _Self& __y) _GLIBCXX_NOEXCEPT
      { return __x._M_node == __y._M_node; }

#if __cpp_impl_three_way_comparison < 201907L
      _GLIBCXX_NODISCARD
      friend bool
      operator!=(const _Self& __x, const _Self& __y) _GLIBCXX_NOEXCEPT
      { return __x._M_node != __y._M_node; }
#endif

      // The only member points to the %list element.
      __detail::_List_node_base* _M_node;
    };
```

## p131

As for the data structure of `list`, it is similar to `vector`. `std::list` is inherited from `std::_List_base`, and `std::_List_base` has a struct called `_List_impl` which contains the real implementation of `list`.

`.../bits/stl_list.h[450]`
```cpp
      struct _List_impl
      : public _Node_alloc_type
      {
	__detail::_List_node_header _M_node;

	_List_impl() _GLIBCXX_NOEXCEPT_IF(
	    is_nothrow_default_constructible<_Node_alloc_type>::value)
	: _Node_alloc_type()
	{ }

	_List_impl(const _Node_alloc_type& __a) _GLIBCXX_NOEXCEPT
	: _Node_alloc_type(__a)
	{ }

#if __cplusplus >= 201103L
	_List_impl(_List_impl&&) = default;

	_List_impl(_Node_alloc_type&& __a, _List_impl&& __x)
	: _Node_alloc_type(std::move(__a)), _M_node(std::move(__x._M_node))
	{ }

	_List_impl(_Node_alloc_type&& __a) noexcept
	: _Node_alloc_type(std::move(__a))
	{ }
#endif
      };

      _List_impl _M_impl;
```

Notice `_M_node` is of class `_List_node_header` which is inherited from `_List_node_base` adding an attribute `_M_size`, used to record the length of the list. 

## p135

`push_back` has its version of lvalue and rvalue after `c++11`.

`.../bits/stl_list.h[1304]`
```cpp
      void
      push_back(const value_type& __x)
      { this->_M_insert(end(), __x); }

#if __cplusplus >= 201103L
      void
      push_back(value_type&& __x)
      { this->_M_insert(end(), std::move(__x)); }
```

`_M_insert` is using universal reference now:

`.../bits/stl_list.h[1992]`
```cpp
#if __cplusplus < 201103L
      void
      _M_insert(iterator __position, const value_type& __x)
      {
	_Node* __tmp = _M_create_node(__x);
	__tmp->_M_hook(__position._M_node);
	this->_M_inc_size(1);
      }
#else
     template<typename... _Args>
       void
       _M_insert(iterator __position, _Args&&... __args)
       {
	 _Node* __tmp = _M_create_node(std::forward<_Args>(__args)...);
	 __tmp->_M_hook(__position._M_node);
	 this->_M_inc_size(1);
       }
#endif
```

## p136

About `erase`:

`.../bits/list.tcc[152]`
```cpp
  template<typename _Tp, typename _Alloc>
    typename list<_Tp, _Alloc>::iterator
    list<_Tp, _Alloc>::
#if __cplusplus >= 201103L
    erase(const_iterator __position) noexcept
#else
    erase(iterator __position)
#endif
    {
      iterator __ret = iterator(__position._M_node->_M_next);
      _M_erase(__position._M_const_cast());
      return __ret;
    }
```

`.../bits/stl_list.h[2012]`
```cpp
      void
      _M_erase(iterator __position) _GLIBCXX_NOEXCEPT
      {
	this->_M_dec_size(1);
	__position._M_node->_M_unhook();
	_Node* __n = static_cast<_Node*>(__position._M_node);
#if __cplusplus >= 201103L
	_Node_alloc_traits::destroy(_M_get_Node_allocator(), __n->_M_valptr());
#else
	_Tp_alloc_type(_M_get_Node_allocator()).destroy(__n->_M_valptr());
#endif
```

## p142

`list::sort` is merge sort, not quick sort.

## 144

There are two asserts in class `deque`. And the third template argument `BufSiz` has been deleted.

`.../bits/stl_deque.h[787]`
```cpp
  template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
    class deque : protected _Deque_base<_Tp, _Alloc>
    {
#ifdef _GLIBCXX_CONCEPT_CHECKS
      // concept requirements
      typedef typename _Alloc::value_type	_Alloc_value_type;
# if __cplusplus < 201103L
      __glibcxx_class_requires(_Tp, _SGIAssignableConcept)
# endif
      __glibcxx_class_requires2(_Tp, _Alloc_value_type, _SameTypeConcept)
#endif

#if __cplusplus >= 201103L
      static_assert(is_same<typename remove_cv<_Tp>::type, _Tp>::value,
	  "std::deque must have a non-const, non-volatile value_type");
# if __cplusplus > 201703L || defined __STRICT_ANSI__
      static_assert(is_same<typename _Alloc::value_type, _Tp>::value,
	  "std::deque must have the same value_type as its allocator");
# endif
#endif

      typedef _Deque_base<_Tp, _Alloc>			_Base;
      typedef typename _Base::_Tp_alloc_type		_Tp_alloc_type;
      typedef typename _Base::_Alloc_traits		_Alloc_traits;
      typedef typename _Base::_Map_pointer		_Map_pointer;

    public:
      typedef _Tp					value_type;
      typedef typename _Alloc_traits::pointer		pointer;
```

In `deuqe::_Base` which is `_Deque_base<_Tp, _Alloc>`, we have following two snippets:

`.../bits/stl_deque.h[507]`
```cpp
      typedef typename iterator::_Map_pointer _Map_pointer; // _Deque_base::_Map_pointer
```

`.../bits/stl_deque.h[455]`
```cpp
      typedef _Deque_iterator<_Tp, _Tp&, _Ptr>	  iterator; // _Deque_base::iterator
```

`.../bits/stl_deque.h[112]`
```cpp
  template<typename _Tp, typename _Ref, typename _Ptr>
    struct _Deque_iterator
    {
#if __cplusplus < 201103L
      typedef _Deque_iterator<_Tp, _Tp&, _Tp*>		   iterator;
      typedef _Deque_iterator<_Tp, const _Tp&, const _Tp*> const_iterator;
      typedef _Tp*					   _Elt_pointer;
      typedef _Tp**					   _Map_pointer;
#else
    private:
      template<typename _CvTp>
	using __iter = _Deque_iterator<_Tp, _CvTp&, __ptr_rebind<_Ptr, _CvTp>>;
    public:
      typedef __iter<_Tp>				   iterator;
      typedef __iter<const _Tp>				   const_iterator;
      typedef __ptr_rebind<_Ptr, _Tp>			   _Elt_pointer;
      typedef __ptr_rebind<_Ptr, _Elt_pointer>		   _Map_pointer;
#endif
```

After `c++11`, we have `_Map_pointer` as `__ptr_rebind<_Ptr, _Elt_pointer>`.

`.../bits/ptr_traits.h[209]`
```cpp
  template<typename _Tp>
    struct pointer_traits<_Tp*> : __ptr_traits_ptr_to<_Tp*, _Tp>
    {
      /// The pointer type
      typedef _Tp* pointer;
      /// The type pointed to
      typedef _Tp  element_type;
      /// Type used to represent the difference between two pointers
      typedef ptrdiff_t difference_type;
      /// A pointer to a different type.
      template<typename _Up> using rebind = _Up*;
    };

  /// Convenience alias for rebinding pointers.
  template<typename _Ptr, typename _Tp>
    using __ptr_rebind = typename pointer_traits<_Ptr>::template rebind<_Tp>; // return type `_Tp*`
```

So we get `_Map_pointer` = `_Elt_pointer*` = `_Tp**`, just as `gcc2.9` in the book. But there are some check makes it legal.

## p146

Here are the attributes of `_Deque_iterator`.

`.../bits/stl_deque.h[142]`
```cpp
      _Elt_pointer _M_cur;
      _Elt_pointer _M_first;
      _Elt_pointer _M_last;
      _Map_pointer _M_node;
```

## p146

The calculation of buffer size is basically same as the old version of `gcc`.

`.../bits/stl_deque.h[131]`
```cpp
      static size_t _S_buffer_size() _GLIBCXX_NOEXCEPT
      { return __deque_buf_size(sizeof(_Tp)); }
```

`.../bits/stl_deque.h[91]`
```cpp
#ifndef _GLIBCXX_DEQUE_BUF_SIZE
#define _GLIBCXX_DEQUE_BUF_SIZE 512
#endif

  _GLIBCXX_CONSTEXPR inline size_t
  __deque_buf_size(size_t __size)
  { return (__size < _GLIBCXX_DEQUE_BUF_SIZE
	    ? size_t(_GLIBCXX_DEQUE_BUF_SIZE / __size) : size_t(1)); }
```

## p151

So as `vector` and `list`, the attributes of `deque` is defined in a struct of `_Deque_base`:

`.../bits/stl_deque.h[509]`
```cpp
      struct _Deque_impl_data
      {
	_Map_pointer _M_map;
	size_t _M_map_size;
	iterator _M_start;
	iterator _M_finish;
```

## p153

One of the constructor of `deque`:

`.../bits/stl_deque.h[890]`
```cpp
      deque(size_type __n, const value_type& __value,
	    const allocator_type& __a = allocator_type())
      : _Base(__a, _S_check_init_len(__n, __a))
      { _M_fill_initialize(__value); }
```

`.../bits/deque.tcc[391]`
```cpp
  template <typename _Tp, typename _Alloc>
    void
    deque<_Tp, _Alloc>::
    _M_fill_initialize(const value_type& __value)
    {
      _Map_pointer __cur;
      __try
	{
	  for (__cur = this->_M_impl._M_start._M_node;
	       __cur < this->_M_impl._M_finish._M_node;
	       ++__cur)
	    std::__uninitialized_fill_a(*__cur, *__cur + _S_buffer_size(),
					__value, _M_get_Tp_allocator());
	  std::__uninitialized_fill_a(this->_M_impl._M_finish._M_first,
				      this->_M_impl._M_finish._M_cur,
				      __value, _M_get_Tp_allocator());
	}
      __catch(...)
	{
	  std::_Destroy(this->_M_impl._M_start, iterator(*__cur, __cur),
			_M_get_Tp_allocator());
	  __throw_exception_again;
	}
    }
```

`.../bits/stl_deque.h[466]`
```cpp
      _Deque_base(const allocator_type& __a, size_t __num_elements)
      : _M_impl(__a)
      { _M_initialize_map(__num_elements); }
```

`.../bits/stl_deque.h[636]`
```cpp
  template<typename _Tp, typename _Alloc>
    void
    _Deque_base<_Tp, _Alloc>::
    _M_initialize_map(size_t __num_elements)
    {
      const size_t __num_nodes = (__num_elements / __deque_buf_size(sizeof(_Tp))
				  + 1);

      this->_M_impl._M_map_size = std::max((size_t) _S_initial_map_size, // enum { _S_initial_map_size = 8 };
					   size_t(__num_nodes + 2));
      this->_M_impl._M_map = _M_allocate_map(this->_M_impl._M_map_size);

      // For "small" maps (needing less than _M_map_size nodes), allocation
      // starts in the middle elements and grows outwards.  So nstart may be
      // the beginning of _M_map, but for small maps it may be as far in as
      // _M_map+3.

      _Map_pointer __nstart = (this->_M_impl._M_map
			       + (this->_M_impl._M_map_size - __num_nodes) / 2);
      _Map_pointer __nfinish = __nstart + __num_nodes;

      __try
	{ _M_create_nodes(__nstart, __nfinish); }
      __catch(...)
	{
	  _M_deallocate_map(this->_M_impl._M_map, this->_M_impl._M_map_size);
	  this->_M_impl._M_map = _Map_pointer();
	  this->_M_impl._M_map_size = 0;
	  __throw_exception_again;
	}

      this->_M_impl._M_start._M_set_node(__nstart);
      this->_M_impl._M_finish._M_set_node(__nfinish - 1);
      this->_M_impl._M_start._M_cur = _M_impl._M_start._M_first;
      this->_M_impl._M_finish._M_cur = (this->_M_impl._M_finish._M_first
					+ __num_elements
					% __deque_buf_size(sizeof(_Tp)));
    }
```

## p156

`push_back` also has two versions, which are for lvalue and rvalue respectively.

`.../bits/stl_deque.h[1537]`
```cpp
      void
      push_back(const value_type& __x)
      {
	if (this->_M_impl._M_finish._M_cur
	    != this->_M_impl._M_finish._M_last - 1)
	  {
	    _Alloc_traits::construct(this->_M_impl,
				     this->_M_impl._M_finish._M_cur, __x);
	    ++this->_M_impl._M_finish._M_cur;
	  }
	else
	  _M_push_back_aux(__x);
      }

#if __cplusplus >= 201103L
      void
      push_back(value_type&& __x)
      { emplace_back(std::move(__x)); }
```

The first version is just like the old version in `gcc2.9` with some function name changed. And notice that the argument of `_M_push_back_aus` is a universal reference now, so that `emplace_back` can also call it.

`emplace_back` is as follows:

`.../bits/deque.tcc[157]`
```cpp
  template<typename _Tp, typename _Alloc>
    template<typename... _Args>
#if __cplusplus > 201402L
      typename deque<_Tp, _Alloc>::reference
#else
      void
#endif
      deque<_Tp, _Alloc>::
      emplace_back(_Args&&... __args)
      {
	if (this->_M_impl._M_finish._M_cur
	    != this->_M_impl._M_finish._M_last - 1)
	  {
	    _Alloc_traits::construct(this->_M_impl,
				     this->_M_impl._M_finish._M_cur,
				     std::forward<_Args>(__args)...);
	    ++this->_M_impl._M_finish._M_cur;
	  }
	else
	  _M_push_back_aux(std::forward<_Args>(__args)...);
#if __cplusplus > 201402L
	return back();
#endif
      }
#endif
```

## p157

The change in `push_front` is mostly the same as the change in `push_back`.

## 

## Summary 

- Restrutured the class with struct `impl`

- Allocator changes, make `std::vector<int, __gnu_cxx::malloc_allocator<double>>` legal

- `rebind`, Lots of `tpyedef` changes. Use `pointer` instead `iterator` to declare data members

- Constructor changes

- `push_back` parameter and implement change

- `static_assert` in `deque`

