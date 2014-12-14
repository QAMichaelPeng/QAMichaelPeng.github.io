---
title: make a shared_ptr from scratch
layout: post
---
shared_ptr is a smart pointer since c++ 11 that will release you from managing the life cycle of objects shared among lots of components without an explicit owner. You can learn it from [cplusplus.com](http://www.cplusplus.com/reference/memory/shared_ptr/). For more details and practice, [boost](http://www.boost.org/doc/libs/1_57_0/libs/smart_ptr/shared_ptr.htm) is a great resource.


Let's consider an interesting question: how to build a shared_ptr from the scratch.

Reference counter
=================

* You must have some mechanism to manage the reference count.
* The reference count operation should be thread safe.
* The reference count instance should be shared between several copies of shared_ptr, so it should be aggregated as a pointer, not composed in a shared_ptr.
* Reference count object should destroy the object if needed. This makes the code of shared_ptr simple.

Here comes the reference count class:
{% highlight c++ linenos tabsize=4 %}
// thread safe inc
inline long atomic_inc(long volatile* addend) {
	return InterlockedIncrement(addend);
}

// thread safe dec
inline long atomic_dec(long volatile* addend) {
	return InterlockedDecrement(addend);
}

// Base class of ref count, handle all the logic of ref count
class RefCountBase {
	long ref_count_ = 1;
public:

	RefCountBase(){
	}

	RefCountBase(const RefCountBase&) = delete;

	RefCountBase& operator=(const RefCountBase&) = delete;

	virtual ~RefCountBase(){
	}

	void Incref() {
		atomic_inc(&ref_count_);
	}

	void Decref() {
		if (atomic_dec(&ref_count_) == 0) {
			Destroy();
			// when refcount == 0. delete the object itself
			// so the owner needn't worry about when to releasing it
			DeleteThis();
		}
	}

	long UseCount() const {
		return ref_count_;
	}

	// destroy the resource
	virtual void Destroy() = 0;

	// destroy the object by itselfs
	virtual void DeleteThis() = 0;
};

// RefCount for pointer delete, use delete operator
// the most common ref count
template <class T>
class RefCount : public RefCountBase {
	T* ptr_;
public:
	RefCount(T* ptr) :ptr_(ptr) {

	}

	RefCount(const RefCount&) = delete;

	RefCount& operator=(RefCount&) = delete;

	virtual ~RefCount(){}

	virtual void Destroy() {
		delete ptr_;
		ptr_ = nullptr;
	}

	virtual void DeleteThis() {
		delete this;
	}
};
{% endhighlight%}
Here we left Destroy and DeleteThis as pure virtual function in RefCountBase and implement it in RefCount. Because share_ptr can not only hold a object allocated by new, but also other resource that need a special destructor.
Here's the shared_ptr class.
{% highlight c++ linenos tabsize=4 %}
// shared_ptr
// shared_ptr<string> p(new string("abc"));
// shared_ptr<string> p2(p);
template <class T>
class shared_ptr {
private:
	RefCountBase* pref_;
	T* ptr_;
public:
	shared_ptr(): pref_(nullptr), ptr_(nullptr) {
	}

	shared_ptr(nullptr_t) :pref_(nullptr), ptr_(nullptr) {
	}

	template<class U> explicit shared_ptr(U* ptr): pref_(new RefCount<T>(ptr)), ptr_(ptr) {
	}

	
	shared_ptr(const shared_ptr& rhs) {
		reset(rhs.ptr_, rhs.pref_);
	}
	
	template<class U> shared_ptr(const shared_ptr<U>& rhs) {
		reset(rhs.ptr_, rhs.pref_);
	}
	
	shared_ptr(shared_ptr&& rhs): pref_(rhs.pref_), ptr_(rhs.ptr_) {
		rhs.pref_ = nullptr;
		rhs.ptr_ = nullptr;
	}
	
	template <class U> shared_ptr(shared_ptr<U>&& rhs) : pref_(rhs.pref_), ptr_(rhs.ptr_) {
		rhs.pref_ = nullptr;
		rhs.ptr_ = nullptr;
	}
	
	shared_ptr& operator=(const shared_ptr& rhs) {
		reset(rhs.ptr_, rhs.pref_);
		return *this;
	}
	
	template<class U>
	shared_ptr& operator=(const shared_ptr<U>& rhs) {
		reset(rhs.ptr_, rhs.pref_);
		return *this;
	}
	
	shared_ptr& operator=(shared_ptr&& rhs) {
		std::swap(pref_, rhs.pref_);
		std::swap(ptr_, rhs.ptr_);
		return *this;
	}
	
	template <class U>
	shared_ptr& operator=(shared_ptr<U>&& rhs) {
		// can't swap internal fields directly
		// assigning from U to T is one way
		shared_ptr(std::move(rhs)).swap(*this);
		return *this;
	}
	
	~shared_ptr() {
		Decref();
	}
	
	void reset() {
		reset(nullptr, nullptr);
	}
	
	T& operator*() const {
		return *(get());
	}
	
	T* get() const {
		return ptr_;
	}
	
	T* operator->() const {
		return get();
	}
	
	void swap(shared_ptr& x) {
		std::swap(ptr_, x.ptr_);
		std::swap(pref_, x.pref_);
	}
	
	long use_count() const {
		if (!pref_) {
			return 0;
		}
		return pref_->UseCount();
	}

	bool unique() const {
		return use_count() == 1;
	}

	explicit operator bool() const {
		return get() != nullptr;
	}

	template <class U> bool owner_before(shared_ptr<U>& rhs) {
		return ptr_ < rhs.ptr_;
	}
private:
	void reset(T* ptr, RefCountBase* refCount) {
		if (pref_) {
			pref_->Decref();
		}
		ptr_ = ptr;
		pref_ = refCount;
		if (refCount) {
			refCount->Incref();
		}
	}
	
	void Decref() {
		if (pref_) {
			pref_->Decref();
			pref_ = nullptr;
		}
	}

	template <class U>
		friend class shared_ptr;
};
{% endhighlight%}
Customized deleter
==================

To support customize deleter, we should add another class RefCountDel whose constructor accept a second argument deletor.
{% highlight c++ linenos tabsize=4 %}
// handle resources with deleter
template <class T, class D>
class RefCountDel: public RefCountBase {
private:
	T* ptr_;
	D del_;
public:
	RefCountDel(T* ptr, D del) : ptr_(ptr), del_(del) {}

	RefCountDel(const RefCountDel&) = delete;

	RefCountDel& operator=(const RefCountDel&) = delete;

	virtual void DeleteThis() {
		delete this;
	}

	virtual void Destroy() {
		del_(ptr_);
	}
};

template <class T> shared_ptr {
	// ...
	template<class U, class D> shared_ptr(U* ptr, D del) : pref_(new RefCountDel<U, D>(ptr, del)), ptr_(ptr) {
	}

	template<class D> shared_ptr(nullptr_t, D del) : pref_(new RefCountDel<T, D>(nullptr, del)), ptr_(nullptr) {
	}
	// ...
};
{% endhighlight %}

Array support
=============

shared_ptr in C++ 11 doesn't support array. We can extend our shared_ptr to support it. To support array, we should create shared_ptr with a customized deleter, like this:
{% highlight c++ linenos tabsize=4 %}
shared_ptr<int> p(new int[10], std::default_delete<int[]>());
{% endhighlight %}
But the above code looks a bit ugly. Since new array is used widely, we should try to elimate the second deleter argument.

To fix it, we have following ways:

1. Create a specialization copy of sahred_ptr for array. This would work, but will introduce redundant codes.
2. Don't create the RefCount class directly with new in shared_ptr. Instead create it with a template function that has a specialization copy for array.

Here is the solution.

{% highlight c++ linenos tabsize=4 %}
template <class T, class U>
RefCountBase* CreateRefCount(shared_ptr<T>* /* for template parameter deduction */, U* p) {
	return new RefCount<T>(p);
}

template <class T>
RefCountBase* CreateRefCount(shared_ptr<T[]>* /* for template parameter deduction */, T* p) {
	return new RefCountDel<T, std::default_delete<T[]>>(p, std::default_delete<T[]>());
}

template <class T>
struct sp_element
{
	typedef T type;
};

template <class T>
struct sp_element<T[]>
{
	typedef T type;
};

template <class T> shared_ptr {
	// if we use T* here and T is array type like int[], we'll have trouble
	typedef typename sp_element<T>::type element_type;
	RefCountBase* pref_;
	element_type* ptr_;
public
	explicit shared_ptr(element_type* ptr) : ptr_(ptr) {
		pref_ = CreateRefCount(this, ptr);
	}

	template<class U> explicit shared_ptr(U* ptr): ptr_(ptr) {
		pref_ = CreateRefCount(this, ptr);
	}
private:
	template<class U>
	void reset(U* ptr, RefCountBase* refCount) {
		if (pref_) {
			pref_->Decref();
		}
		ptr_ = ptr;
		pref_ = refCount;
		if (refCount) {
			refCount->Incref();
		}
	}
	...
}
{% endhighlight %}

Alias constructor
=================

In shared_ptr we stored two pointers, one is the stored pointer ptr_, one is the ref count pointer pref_. Since pref_ has a copy of the stored pointer, why we store another copy in the shared_ptr object? This is for alias consturector. 
Think about a scenario: you have a big object a in type A allocated by new, and it has a small field b with type B. You want to keep a shared_ptr for b, how can you do this?

Since b is owned by a, you shouldn't delete it by yourself with a shared_ptr. You can use it like:
{% highlight c++ linenos tabsize=4 %}
shared_ptr<A> a(new A);
a.b->func(...)
{% endhighlight %}

But a.b->func(...) is not convenient as a->func(...). And you can't keep a raw pointer to a.a without lifecycle managment. If a is destroyed, pointer to a.a will be dangling.

Here comes alias constructor
{% highlight c++ linenos tabsize=4 %}
shared_ptr<A> a(new A);
shared_ptr<B> b(a, &a.b);
b->func(...)
{% endhighlight %}

For more details. please see [shared_ptr aliasing constructor](http://www.codesynthesis.com/~boris/blog/2012/04/25/shared-ptr-aliasing-constructor/).

Below is the implement:

{% highlight c++ linenos tabsize=4 %}
// aliasing
template <class U> shared_ptr(const shared_ptr<U>& rhs, element_type* ptr):ptr_(ptr) {
	pref_ = rhs.pref_;
	if (pref_) {
		pref_->Incref();
	}
}
{% endhighlight %}

make_shared
===========

So far everything is OK, except that to allocate an object with an shared_ptr, we need to new, one for the object itself, and another is for the refCount. How can we use only one memory allocation to allocate the two object? One idea is to put the stored object as a field of ref count object. We can know the object size at compile time, but how can we initialize it in the runtime with the infinite possibilities of constructor? Fortunatelly c++ 11 brings us [variadic template](http://en.wikipedia.org/wiki/Variadic_template) and [perfect forwarding](http://en.cppreference.com/w/cpp/utility/forward). So we can use (placement new)[http://en.wikipedia.org/wiki/Placement_syntax] to initialize the stored object with arguments we don't know in the templates.
{% highlight c++ linenos tabsize=4 %}
template <class T>
class RefCountObj: public RefCountBase {
private:
    typename std::aligned_storage<sizeof(T)>::type storage_;
public:
    template<class... Types>
    RefCountObj(Types... args) {
        new (&storage_)T(std::forward<Types>(args)...);
    }

    T* Get() const {
        return (T*)(&storage_);
    }

    virtual void Destroy() {
        Get()->~T();
    }

    virtual void DeleteThis() {
        delete this;
    }
};

template <class T, class... Types>
shared_ptr<T> make_shared(Types&&... args) {
    RefCountObj<T>* obj = new RefCountObj<T>(std::forward<Types>(args)...);
    shared_ptr<T> p;
    p.reset_ref_0(obj->Get(), obj);
    return p;
}
{% endhighlight %}

Code list
=========

Here's the full code list:

{% highlight c++ linenos tabsize=4 %}
#pragma once
#ifdef _WIN32
#include <windows.h>
#endif
#include <algorithm>
#include <cstddef>
#include <memory>

namespace mp {
	template <class T> class shared_ptr;
	// thread safe inc
	inline long atomic_inc(long volatile* addend) {
		return InterlockedIncrement(addend);
	}

	// thread safe dec
	inline long atomic_dec(long volatile* addend) {
		return InterlockedDecrement(addend);
	}

	// Base class of ref count, handle all the logic of ref count
	class RefCountBase {
		long ref_count_ = 1;
	public:
		RefCountBase(){
		}
		RefCountBase(const RefCountBase&) = delete;
		RefCountBase& operator=(const RefCountBase&) = delete;

		virtual ~RefCountBase(){
		}

		void Incref() {
			atomic_inc(&ref_count_);
		}
		void Decref() {
			if (atomic_dec(&ref_count_) == 0) {
				Destroy();
				// when refcount == 0. delete the object itself
				// so the owner needn't worry about when to releasing it
				DeleteThis();
			}
		}
		long UseCount() const {
			return ref_count_;
		}
		// destroy the resource
		virtual void Destroy() = 0;
		// destroy the object by itselfs
		virtual void DeleteThis() = 0;
	};

	// RefCount for pointer delete, use delete operator
	// the most common ref count
	template <class T>
	class RefCount : public RefCountBase {
		T* ptr_;
	public:
		RefCount(T* ptr) :ptr_(ptr) {

		}

		RefCount(const RefCount&) = delete;

		RefCount& operator=(RefCount&) = delete;

		virtual ~RefCount(){}

		virtual void Destroy() {
			delete ptr_;
			ptr_ = nullptr;
		}

		virtual void DeleteThis() {
			delete this;
		}
	};

    template <class T, class D>
    class RefCountDel: public RefCountBase {
    private:
        T* ptr_;
        D del_;
    public:
        RefCountDel(T* ptr, D del): ptr_(ptr), del_(del) {
        }

        RefCountDel(const RefCountDel&) = delete;

        RefCountDel& operator=(const RefCountDel&) = delete;

        virtual void Destroy() {
            del_(ptr_);
        }

        virtual void DeleteThis() {
            delete this;
        }
    };

    template <class T>
    class RefCountObj: public RefCountBase {
    private:
        typename std::aligned_storage<sizeof(T)>::type storage_;
    public:
        template<class... Types>
        RefCountObj(Types... args) {
            new (&storage_)T(std::forward<Types>(args)...);
        }

        T* Get() const {
            return (T*)(&storage_);
        }

        virtual void Destroy() {
            Get()->~T();
        }

        virtual void DeleteThis() {
            delete this;
        }
    };

    template <class T, class... Types>
    shared_ptr<T> make_shared(Types&&... args) {
        RefCountObj<T>* obj = new RefCountObj<T>(std::forward<Types>(args)...);
        shared_ptr<T> p;
        p.reset_ref_0(obj->Get(), obj);
        return p;
    }

	template <class T, class U>
	RefCountBase* CreateRefCount(shared_ptr<T>* /* for template parameter deduction */, U* p) {
		return new RefCount<T>(p);
	}

	template <class T>
	RefCountBase* CreateRefCount(shared_ptr<T[]>* /* for template parameter deduction */, T* p) {
		return new RefCountDel<T, std::default_delete<T[]>>(p, std::default_delete<T[]>());
	}

	template <class T>
	struct sp_element
	{
		typedef T type;
	};

	template <class T>
	struct sp_element<T[]>
	{
		typedef T type;
	};

	// shared_ptr
	// shared_ptr<string> p(new string("abc"));
	// shared_ptr<string> p2(p);
	template <class T>
	class shared_ptr {
	private:
		// if we use T* here and T is array type like int[], we'll have trouble
		typedef typename sp_element<T>::type element_type;
		RefCountBase* pref_;
		element_type* ptr_;
	public:
		shared_ptr() : pref_(nullptr), ptr_(nullptr) {
		}

		shared_ptr(element_type* ptr) : ptr_(ptr) {
			pref_ = CreateRefCount(this, ptr);
		}

		template<class U> explicit shared_ptr(U* ptr) :
			pref_(CreateRefCount(this, ptr)), ptr_(ptr) {
		}

		template<class U, class D> shared_ptr(U* ptr, D del) :
			pref_(new RefCountDel<U, D>(ptr, del)), ptr_(ptr) {
		}

		shared_ptr(nullptr_t) :pref_(nullptr), ptr_(nullptr) {
		}

        template<class D> shared_ptr(nullptr_t, D del) : 
            pref_(new RefCountDel<T, D>(nullptr, del)), ptr_(nullptr) {
        }

		template<class U> shared_ptr(const shared_ptr<U>& u, T* ptr):
			pref_(nullptr), ptr_(nullptr){
			reset_ref(ptr, u.pref_);
		}
        
		shared_ptr(const shared_ptr& rhs):
			pref_(nullptr), ptr_(nullptr) {
			reset_ref(rhs.ptr_, rhs.pref_);
		}

		template<class U> shared_ptr(const shared_ptr<U>& rhs): 
            ptr_(nullptr), pref_(nullptr) {
			reset_ref(rhs.ptr_, rhs.pref_);
		}

        // move ctor
		shared_ptr(shared_ptr&& rhs): 
            pref_(rhs.pref_), ptr_(rhs.ptr_) {
			rhs.pref_ = nullptr;
			rhs.ptr_ = nullptr;
		}
		template <class U> shared_ptr(shared_ptr<U>&& rhs) : pref_(rhs.pref_), ptr_(rhs.ptr_) {
			rhs.pref_ = nullptr;
			rhs.ptr_ = nullptr;
		}
		
		shared_ptr& operator=(const shared_ptr& rhs) {
			reset_ref(rhs.ptr_, rhs.pref_);
			return *this;
		}

		shared_ptr& operator=(shared_ptr&& rhs) {
			reset_ref(rhs.ptr_, rhs.pref_);
			return *this;
		}

		template <class U>
		shared_ptr& operator=(shared_ptr<U>&& rhs) {
			reset_ref(rhs.ptr_, rhs.pref_);
			return *this;
		}

		~shared_ptr() {
			Decref();
		}
		void reset() {
			reset_ref(nullptr, nullptr);
		}

		template <class U>
		void reset(U* ptr) {
			shared_ptr<T>(ptr).swap(*this);
		}

		template< class U, class D>
		void reset(U* ptr, D del) {
			shared_ptr<T>(ptr, del).swap(*this);
		}

		T& operator*() const {
			return *(get());
		}
		T* get() const {
			return ptr_;
		}
		T* operator->() const {
			return get();
		}

		void swap(shared_ptr& x) {
			std::swap(ptr_, x.ptr_);
			std::swap(pref_, x.pref_);
		}
		long use_count() const {
			if (!pref_) {
				return 0;
			}
			return pref_->UseCount();
		}
		bool unique() const {
			return use_count() == 1;
		}
		explicit operator bool() const {
			return get() != nullptr;
		}
		template <class U> bool owner_before(shared_ptr<U>& rhs) {
			return pref_ < rhs.pref_;
		}
	private:
        // reset without incref
        void reset_ref_0(element_type* ptr, RefCountBase* refCount) {
			if (pref_) {
				pref_->Decref();
			}
			ptr_ = ptr;
			pref_ = refCount;

        }
		void reset_ref(element_type* ptr, RefCountBase* refCount) {
			if (refCount) {
				refCount->Incref();
			}
            reset_ref_0(ptr, refCount);
		}
		

		void Decref() {
			if (pref_) {
				pref_->Decref();
				pref_ = nullptr;
			}
		}

        // friends
		template <class U> friend class shared_ptr;
        template <class U, class... Types> friend shared_ptr<U> make_shared(Types&&... args);
	};

    template <class T>
    void swap(shared_ptr<T>& lhs, shared_ptr<T>& rhs) {
        lhs.swap(rhs);
    }

    template < class T, class U > 
    bool operator==(const shared_ptr<T>& lhs, const shared_ptr<U>& rhs) {
        return lhs.get() == rhs.get();
    }

    template< class T, class U > 
    bool operator!=(const shared_ptr<T>& lhs, const shared_ptr<U>& rhs) {
        return lhs.get() != rhs.get();
    }

    template< class T, class U > 
    bool operator<(const shared_ptr<T>& lhs, const shared_ptr<U>& rhs) {
        return lhs.get() < rhs.get();
    }
    template< class T, class U > 
    bool operator>(const shared_ptr<T>& lhs, const shared_ptr<U>& rhs) {
        return lhs.get() > rhs.get();
    }
    template< class T, class U > 
    bool operator<=(const shared_ptr<T>& lhs, const shared_ptr<U>& rhs) {
        return lhs.get() <= rhs.get();
    }

    template< class T, class U > 
    bool operator>=(const shared_ptr<T>& lhs, const shared_ptr<U>& rhs) {
        return lhs.get() >= rhs.get();
    }

    template< class T > 
    bool operator==(const shared_ptr<T>& lhs, std::nullptr_t) {
        return lhs.get() == nullptr;
    }

    template< class T >
    bool operator==(std::nullptr_t, const shared_ptr<T>& rhs) {
        return nullptr == rhs.get();
    }

    template< class T >
    bool operator!=(const shared_ptr<T>& lhs, std::nullptr_t) {
        return lhs.get() != nullptr;
    }

    template< class T >
    bool operator!=(std::nullptr_t, const shared_ptr<T>& rhs) {
        return nullptr != rhs.get();
    }

    template< class T >
    bool operator<(const shared_ptr<T>& lhs, std::nullptr_t) {
        return lhs.get() < nullptr;
    }

    template< class T >
    bool operator<(std::nullptr_t, const shared_ptr<T>& rhs) {
        return nullptr < rhs.get();
    }

    template< class T >
    bool operator<=(const shared_ptr<T>& lhs, std::nullptr_t) {
        return lhs.get() <= nullptr;
    }

    template< class T >
    bool operator<=(std::nullptr_t, const shared_ptr<T>& rhs) {
        return nullptr <= rhs.get();
    }

    template< class T >
    bool operator>(const shared_ptr<T>& lhs, std::nullptr_t) {
        return lhs.get() > nullptr;
    }

    template< class T >
    bool operator>(std::nullptr_t, const shared_ptr<T>& rhs) {
        return nullptr > rhs.get();
    }

    template< class T >
    bool operator>=(const shared_ptr<T>& lhs, std::nullptr_t) {
        return lhs.get() >= nullptr;
    }

    template< class T >
    bool operator>=(std::nullptr_t, const shared_ptr<T>& rhs) {
        return nullptr >= rhs.get();
    }
}

{% endhighlight %}

Here's the test code.

{% highlight c++ linenos tabsize=4 %}
#include "stdafx.h"
#include "smartptr.h"

namespace {
	using namespace mp;
	class B {
	public:
		int value_ = 0;
		int Inc() {
			return ++value_;
		}
	};

	class A
	{
	public:
		static std::string ctor_dtor_recorder_;
		void ClearCtorDtorRecorder() {
			A::ctor_dtor_recorder_.clear();
		}
		int value_;
		B b_;
		A() {
			value_ = 0;
			ctor_dtor_recorder_.push_back('C');
		}
		
		A(int val) : value_(val) {
			ctor_dtor_recorder_.push_back('C');
		}

		int Inc() {
			return ++value_;
		}
		~A() {
			ctor_dtor_recorder_.push_back('D');
		}
	};

	std::string A::ctor_dtor_recorder_;

	class A2: public A {
	};

    class SharedPtrTest: public testing::Test {
        protected:
            virtual void SetUp() {
                A::ctor_dtor_recorder_.clear();
            }
    };

	#define AssertCtorDtor(expected) ASSERT_EQ((expected), A::ctor_dtor_recorder_)
	
	TEST_F(SharedPtrTest, ctor) {
		A::ctor_dtor_recorder_.clear();
		{
			A* a = new A();
			shared_ptr<A> p(a);
		}
		AssertCtorDtor("CD");
	}

	TEST_F(SharedPtrTest, ctor_array_deleter) {
		int i = 0;
		auto x = [&i](A* a1) {++i; delete[] a1; };
		mp::shared_ptr<A[]> a(new A[5], x);
		AssertCtorDtor("CCCCC");
		a.reset();
		AssertCtorDtor("CCCCCDDDDD");
	}

	TEST_F(SharedPtrTest, ctor_array) {
		mp::shared_ptr<A[]> a(new A[5]);
		AssertCtorDtor("CCCCC");
		a.reset();
		AssertCtorDtor("CCCCCDDDDD");
	}

	TEST_F(SharedPtrTest, nullptr) {
		shared_ptr<A> p(nullptr);
		ASSERT_EQ(nullptr, p.get());
	}

	TEST_F(SharedPtrTest, nullptr_deleter) {
		int i = 0;
		auto x = [&i](A* p1){++i; delete p1; };
		{
			shared_ptr<A> p(nullptr, x);
			ASSERT_EQ(nullptr, p.get());
		}
		ASSERT_EQ(1, i);
		AssertCtorDtor("");
	}

	TEST_F(SharedPtrTest, ctor_aliasing) {
		auto pa = new A();
		shared_ptr<A> a(pa);
		shared_ptr<B> b(a, &a->b_);
		ASSERT_EQ(2, b.use_count());
		a.reset();
		AssertCtorDtor("C");
		b->Inc();
		ASSERT_EQ(1, pa->b_.value_);
		b.reset();
		AssertCtorDtor("CD");
	}

	TEST_F(SharedPtrTest, copy_ctor) {
		{
			A* a = new A();
			shared_ptr<A> p(a);
			shared_ptr<A> p2(p);
			p2->Inc();
			ASSERT_EQ(1, a->value_);
		}
		AssertCtorDtor("CD");
	}

	TEST_F(SharedPtrTest, ctor_deletor) {
		int i = 0;
		auto x = [&i](A* p) {++i; delete p; };
		{
			shared_ptr<A> p(new A(), x);
		}
		AssertCtorDtor("CD");
		ASSERT_EQ(1, i);
	}

	TEST_F(SharedPtrTest, ctor_difftype) {
		{
			A2* a = new A2();
			shared_ptr<A> p(a);
		}
		AssertCtorDtor("CD");
	}

	TEST_F(SharedPtrTest, copy_ctor_difftype) {
		{
			A2* a = new A2();
			shared_ptr<A2> p(a);
			shared_ptr<A> p2(p);
			p2->Inc();
			ASSERT_EQ(1, a->value_);
		}
		AssertCtorDtor("CD");
	}

	TEST_F(SharedPtrTest, move_ctor) {
		A* a = new A();
		shared_ptr<A> p1(a);
		shared_ptr<A> p2(std::move(p1));
		ASSERT_EQ(nullptr, p1.get());
		ASSERT_TRUE(p2.unique());
	}

	TEST_F(SharedPtrTest, move_ctor_difftype) {
		A2* a = new A2();
		shared_ptr<A2> p1(a);
		shared_ptr<A> p2(std::move(p1));
		ASSERT_EQ(nullptr, p1.get());
		ASSERT_EQ(a, p2.get());
	}

	TEST_F(SharedPtrTest, assign_op) {
		shared_ptr<A> p0;
		{
			A* a = new A();
			shared_ptr<A> p(a);
			p0 = p;
		}
		AssertCtorDtor("C");
	}

	TEST_F(SharedPtrTest, reset) {
		shared_ptr<A> p(new A());
		AssertCtorDtor("C");
		p.reset();
		AssertCtorDtor("CD");
	}

	TEST_F(SharedPtrTest, reset_difftype) {
		shared_ptr<A> p(new A());
		AssertCtorDtor("C");
		p.reset(new A2());
		AssertCtorDtor("CCD");
	}

	TEST_F(SharedPtrTest, reset_difftype_deletor) {
		shared_ptr<A> p(new A());
		AssertCtorDtor("C");
		int i = 0;
		auto x = [&i](A* p1) {++i; delete p1; };
		p.reset(new A2(), x);
		AssertCtorDtor("CCD");
		p.reset();
		AssertCtorDtor("CCDD");
	}

	TEST_F(SharedPtrTest, get) {
		A* a = new A();
		shared_ptr<A> p(a);
		ASSERT_EQ(a, p.get());
		p.reset();
		ASSERT_EQ(nullptr, p.get());
	}

	TEST_F(SharedPtrTest, arrow) {
		A* a = new A();
		shared_ptr<A> p(a);
		p->Inc();
		ASSERT_EQ(1, a->value_);
	}

	TEST_F(SharedPtrTest, deference) {
		A* a = new A();
		shared_ptr<A> p(a);
		(*p).Inc();
		ASSERT_EQ(1, a->value_);
	}

	TEST_F(SharedPtrTest, swap) {
		A* a1 = new A();
		A* a2 = new A();
		shared_ptr<A> p1(a1), p11(p1);
		shared_ptr<A> p2(a2);
		p1.swap(p2);
		ASSERT_EQ(a2, p1.get());
		ASSERT_EQ(a1, p2.get());
		ASSERT_EQ(a1, p11.get());
	}

	TEST_F(SharedPtrTest, use_count) {
		shared_ptr<A> p1(new A());
		ASSERT_EQ(1, p1.use_count());
		shared_ptr<A> p2;
		ASSERT_EQ(0, p2.use_count());
		p2 = p1;
		ASSERT_EQ(2, p1.use_count());
		p2.reset();
		ASSERT_EQ(1, p1.use_count());
		p1.reset();
		ASSERT_EQ(0, p1.use_count());
	}

	TEST_F(SharedPtrTest, unique) {
		shared_ptr<A> p1(new A());
		ASSERT_TRUE(p1.unique());
		shared_ptr<A> p2;
		ASSERT_FALSE(p2.unique());
		p2 = p1;
		ASSERT_FALSE(p1.unique());
		ASSERT_FALSE(p2.unique());
		p2.reset();
		ASSERT_TRUE(p1.unique());
		ASSERT_FALSE(p2.unique());
		p1.reset();
		ASSERT_FALSE(p1.unique());
	}

	TEST_F(SharedPtrTest, bool) {
		shared_ptr<A> p(new A());
		ASSERT_TRUE((bool)p);
		bool v = false;
		if (p) {
			v = true;
		}
	
		ASSERT_TRUE(v);
		p.reset();
		ASSERT_FALSE((bool)p);
	}

	TEST_F(SharedPtrTest, owner_before) {
		A* a1 = new A(), *a2 = new A();
		shared_ptr<A> p1(a1), p2(a2);
		ASSERT_TRUE(p1.owner_before(p2) || p2.owner_before(p1));
		ASSERT_FALSE(p1.owner_before(p2) && p2.owner_before(p1));
		p2 = p1;
		ASSERT_FALSE(p1.owner_before(p2));
		ASSERT_FALSE(p2.owner_before(p1));
	}

	TEST_F(SharedPtrTest, make_shared) {
		shared_ptr<A> a = make_shared<A>(3);
		a->Inc();
		ASSERT_EQ(4, a->value_);
		a = shared_ptr<A>(new A(8));
		ASSERT_EQ(8, a->value_);
		AssertCtorDtor("CCD");
	}
		
	TEST_F(SharedPtrTest, swap_global) {
		A* a1 = new A(), *a2 = new A2();
		shared_ptr<A> p1(a1), p2(a2);
		bool ownerBefore1 = p1.owner_before(p2);
		swap(p1, p2);
		ASSERT_EQ(a2, p1.get());
		ASSERT_EQ(a1, p2.get());
		bool ownerBefore2 = p1.owner_before(p2);
		ASSERT_TRUE(ownerBefore1 || ownerBefore2);
		ASSERT_FALSE(ownerBefore1 && ownerBefore2);
	}

	TEST_F(SharedPtrTest, relation_op) {
		A* a1 = new A();
		A2* a2 = new A2();
		shared_ptr<A> p1(a1);
		shared_ptr<A2> p2(a2);
		ASSERT_TRUE(p1 != p2);
		ASSERT_FALSE(p1 == p2);
		ASSERT_EQ(p1 < p2, a1 < a2);
		ASSERT_EQ(p1 > p2, a1 > a2);
		ASSERT_EQ(p1 <= p2, a1 <= a2);
		ASSERT_EQ(p1 >= p2, a1 >= a2);

		p1 = p2;
		ASSERT_TRUE(p1 == p2);
		ASSERT_FALSE(p1 != p2);
		ASSERT_FALSE(p1 < p2);
		ASSERT_TRUE(p1 <= p2);
		ASSERT_FALSE(p1 > p2);
		ASSERT_TRUE(p1 >= p2);

	
		ASSERT_FALSE(p1 == nullptr);
		ASSERT_FALSE(nullptr == p1);
		ASSERT_TRUE(p1 != nullptr);
		ASSERT_TRUE(nullptr != p1);
		ASSERT_FALSE(p1 < nullptr);
		ASSERT_FALSE(p1 <= nullptr);
		ASSERT_TRUE(nullptr < p1);
        ASSERT_TRUE(nullptr <= p1);
		ASSERT_TRUE(p1 > nullptr);
		ASSERT_TRUE(p1 >= nullptr);
		ASSERT_FALSE(nullptr > p1);
        ASSERT_FALSE(nullptr >= p1);

		p2.reset();
		ASSERT_TRUE(p2 == nullptr);
		ASSERT_TRUE(nullptr == p2);
		ASSERT_FALSE(p2 != nullptr);
		ASSERT_FALSE(nullptr != p2);
		ASSERT_FALSE(p2 < nullptr);
		ASSERT_TRUE(p2 <= nullptr);
		ASSERT_FALSE(nullptr < p2);
        ASSERT_TRUE(nullptr <= p2);
		ASSERT_FALSE(p2 > nullptr);
		ASSERT_TRUE(p2 >= nullptr);
		ASSERT_FALSE(nullptr > p2);
        ASSERT_TRUE(nullptr >= p2);
	}
}
{% endhighlight %}
