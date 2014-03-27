---
layout: post
title: "The Family of Smart Pointers in Gecko"
description: ""
category: Something you should know
tags: [mozilla, gecko, c++]
---
{% include JB/setup %}

Using raw pointer correctly is always a nightmare for C/C++ developers. It's too easy to make mistakes, such as use-after-free and resource leakage.
Auto pointer and share pointer were introduced to solve most of the problem, or at least to reveal the problem at early stage.
C++11 standard defines `unique_ptr` and `shared_ptr` for it, however, Gecko still uses its own specific smart pointer type due to some compiler backward capability issue.

There are at least 6 smart pointer types with similar naming and it is very confusing for people who writing patch for various modules.
This article is trying to give more clue about when to use which smart pointer.

nsCOMPtr
--------
Every class that implements `nsISupports` interface can be consider as an XPCOM object.
Each XPCOM object is reference counted and provides `AddRef()` and `Release()` for controlling the reference counting explictly.
Gecko introduces two smart pointer types with RAII features for maintaining the reference count easily.
nsCOMPtr is probbaly the most common auto pointer in Gecko code base.
It is used to hold a pointer of XPCOM interface type, e.g. `nsISupports`, so it will be used to upcast/downcast to other XPCOM interface via `do_QueryInterface()`
`do_QueryObject()` is used to solve multiple inheritance casting problem while using `do_QueryInterface()` and it'll have extra code size overhead because of using template.

    class nsIBar : public nsISupports {...}; // an XPCOM interface declaration
    class nsFoo : public nsIBar {...}; // implement AddRef/Release/QueryInterfaces as an XPCOM object

    nsCOMPtr<nsISupports> isupportPtr = new nsFoo();
    {
        nsCOMPtr<nsIBar> ibarPtr = do_QueryInterface(isupportPtr); // type casting across different XPCOM interfaces.
        // Release() will be invoked after exiting scope.
    }

    void someGetterFunction(nsIBar**); // function using output parameter
    nsCOMPtr<nsIBar> outputPtr;
    someGetterFunction(getter_AddRefs(outputPtr));

    already_AddRefed<nsIBar> someFunction(); // function return an XPCOM object
    nsCOMPtr<nsIBar> returnPtr = someFunction();

nsRefPtr
--------
nsRefPtr is used to hold a pointer for any type that implements `AddRef()` and `Release()`.
It's often used to hold a reference to a concrete type of XPCOM object.

    class Foo {
      NS_INLINE_DECL_REFCOUNTING(Foo) // helper for implementing AddRef/Release
    };

    nsRefPtr<Foo> fooPtr = new Foo();

    void someGetterFunction(Foo**); // function using output parameter.
    nsRefPtr<Foo> outputPtr;
    someGetterFunction(getter_AddRefs(outputPtr));

    already_AddRefed<Foo> someFunction(); // function return an object.
    nsRefPtr<Foo> returnPtr = someFunction();

RefPtr
------
RefPtr is a reference-counted pointer which defined in MFBT.
Any class that might be used before XPCOM startup should use RefPtr as a shard pointer.
`RefCounted<T>` is an helper class for adding reference counting mechanism to your object.

    class Foo: public RefCounted<Foo> {};

    RefPtr<Foo> fooPtr = new Foo();
    {
        RefPtr<Foo> anotherPtr = fooPtr; // AddRef() invoked, reference count is 2 now.
        // Release() invoked after exiting scope.
    }

    void someGetterFunction(Foo**); // function using output parameter.
    RefPtr<Foo> outputPtr;
    someGetterFunction(byRef(outputPtr));

    TemporaryRef<Foo> someFunction(); // TemporaryRef represents a rvalue of a RefPtr
    RefPtr<Foo> retPtr = someFunction(); // someFunction() = fooPtr; will not compile.

nsAutoRef/nsCountedRef
----------------------
nsAutoRef/nsCountedRef provide auto resource management for anything doesn't have destructor or anything we cannot change the type definition.
By implementing `nsAutoRefTraits<T>`, user can define the behavior while nsAutoRef is releasing the resource of type T.

    class nsAutoRefTraits<Foo> {
        typedef Foo* RawRef;
        static RawRef Void() { return nullptr; }
        static void Release(RawRef aRawRef) {
            // do something about releasing the aRawRef, e.g. close(aRawRef);
        }
        static void AddRef(RawRef aRawRef) {
            // do something about copying reference, e.g. AddReferenceFoo(aRawRef);
        }
    };

    nsAutoRef<Foo> autoRef = new Foo();

    nsReturnRef<Foo> someFunction(); // nsReturnRef represents a rvalue of a nsAutoRef
    nsAutoRef<Foo> returnRef = someFunction();

    nsCountedRef<Foo> countRef1 = new Foo();
    {
        nsCountedRef<Foo> counteRef2 = countRef1; // nsAutoRefTraits<Foo>::AddRef() is invoked.
        // nsAutoRefTraits<Foo>::Release() is invoked after exiting scope.
    }

nsAutoPtr/nsAutoArrayPtr
------------------------
nsAutoPtr is an smart pointer that hold a reference to a resource that allocated by new operator.
Assignment of a nsAutoPtr will transfer the ownership of the resource.
nsAutoPtr cannot be the return value type of a function.
nsAutoArrayPtr is the smart pointer for array that allocated by `new []` operator.

    class Foo {};
    nsAutoPtr<Foo> autoPtr = new Foo();
    nsAutoPtr<Foo> autoPtr2 = autoPtr; // autoPtr no longer point to the Foo object.

    void someGetterFunction(Foo**); // function using output parameter
    nsAutoPtr<Foo> outputPtr;
    someGetterFunction(getter_Transfers(outputPtr));

    nsAutoArrayPtr<uint8_t> iarray = new uint8_t[10]; // delete [] will be invoked after exiting scope
    nsAutoArrayPtr<uint8_t> iarray2 = iarray; // transfer uint8_t[10] from iarray to iarray2

