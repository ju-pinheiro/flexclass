# Basic Concepts

`Flexclass` empowers the user to declare structures and arrays in a single allocation.

To steps are necessary to make a struct be recognized by the library:
- Declare handles that will refer to a sequence of objects in memory
- Declare a `fc_handles` method that the library will use to query

For example:
```
struct MyType
{
    int myInt;
    fc::Array<long> longs;
    std::string str;
    
    auto fc_handles() { return fc::make_tuple(&longs); }
};
```

`fc::Array<long>` is identified as a `Handle`, meaning it wants to point to a group of `long` objects.
So we have the following layout:

```
            ____________________
           |                    V
| [int] [long*] [std::string] | [long] [long] [long] ...
|                             |
|           MyType            | 
```

## Instantiating

The `flexclass` aware type has dynamic size, so it cannot be allocated on the stack. To allocate one on the heap, use the `fc::make_unique` method:

```
auto m = fc::make_unique<MyType>(...)(...);
```

This returns a RAII enabled object, which will cleanup after itself when it's destroyed.

If you prefer to manually manage this object, use `fc::make` and `fc::destroy`:
```
auto* m = fc::make<MyType>(...)(...);
fc::destroy(m);
```

## Arguments

`make` and `make_unique` methods provide a two step initialization and require the syntax:
```
    fc::make<T>(array arguments...)(T constructor arguments);
```
In the first parenthesis the function expects the arguments for creating arrays.
So if `fc_handles` returns `N` handles, then `make` expects `N` arguments.

The arguments for the array creation is the size of the array, but [more avanced forms of initialization](../master/UserGuide.md#handle-initialization) are available.

The second pair of parenthesis take the arguments to create the type `T`.

# Provided Handles

`Flexclass` provides a handful of handles so the user doesn't have to write them by hand.

- `*Array` handles provide only the `begin` iterator for the object sequence
- `*Range` handles provide both `begin`and `end`

Available handles are:
- `fc::Array<T>`: Contains one `T*` to indicate the begin of the object sequence
- `fc::Range<T>`: Contains two `T*` to indicate both begin and end
- `fc::AdjacentArray<T, int Idx = -1>`: Contains no data as it assumes its array is adjacent to the data from handle in `Idx`
    - If `Idx` is `-1`, it assumes the begin of its array is after the type.
- `fc::AdjacentRange<T, int Idx = -1>`: Like `fc::AdjacentArray<T>` but contains a `T*` to also know the end of the object sequence

Note that for `Adjacent*` handles to work, they take a pointer to the type on `begin` and `end` methods:
```
    struct Type
    {
        auto fc_handles() { return fc::make_tuple(&data); }
        
        auto begin() { return data.begin(this); }
        
        fc::AdjacentArray<char> data;
    };
```

# Custom handles

Handles being provided with the library use the framework to implement custom handles.
Here is a customization example of a handle that assumes that the data is very close to the base and is very small:

```
template <class T>
struct NearAndSmallRange : fc::Handle<T> // I know my Range will be very close and very small
{
    // This is called by the library to inform where the array for this handle was placed
    void setLocation(T* begin, T* end)
    {
        auto size = end - begin;
        auto offset = (uintptr_t)begin - (uintptr_t)this;
        
        assert(size   <= std::numeric_limits<std::uint8_t>::max());
        assert(offset <= std::numeric_limits<std::uint8_t>::max());
        
        m_beginOffset = offset;
        m_size = size;
    }

    // This is called by the library when the user asks where is the begin of the array
    // The pointer of the base itself is provided, in case this handle needs information from the base
    //     See the implementation of fc::AdjacentArray
    template <class Base>
    auto begin(const Base* ptr) const
    {
        return (T*)((uintptr_t)this + m_beginOffset);
    }
    
    template <class Base>
    auto end(const Base* ptr) const
    {
        return begin(ptr) + m_size;
    }

    std::uint8_t m_beginOffset, m_size;
}
```

# Handle initialization

For each handle, the user is expected to pass a size of the array that will be allocated for it. Arrays are default initialized, but the user may use `fc::arg` to pass an input iterator that will be called to obtain initial values each element:

```
struct Type
{
    auto fc_handles() { return fc::make_tuple(&data); }      
    fc::Array<int> data;
};

// Create 10 ints
auto m1 = fc::make<Type>(10)();

// Create 10 ints
auto m2 = fc::make<Type>(  fc::arg(10)  )();

// Create 10 initialized ints
std::vector<int> v {1,2,3,4,5,6,7,8,9,10};
auto m3 = fc::make<Type>(  fc::arg(10, v.begin())  )();
```
In this last scenario, the array will contain values `1` to `10` obtained from the iterator.

# Allocators

Construction with custom allocators is also supported. However, `Flexclass` does not store the allocator in the structure like other data structures (`std::vector`, `std::map`, ... ).
Generally there will be a main entity managing all flexclasses, so it knows which allocator to use. Also, adding the allocator as a member can be done over-the-hood, by the user.

To allocate using a custom allocator, invoke `fc::make` with `fc::withAllocator` and your allocator in the array argument list:
```
struct Type
{
    auto fc_handles() { return fc::make_tuple(&data); }      
    fc::Array<int> data;
};

MyAllocator alloc;
auto fclass = fc::make<Type>(fc::withAllocator, alloc, 10)();
...
fc::destroy(fclass, alloc);
```

# Exception Guarantees

`Flexclass` is well behaved with respect to lifetimes and exceptions. That means all objects created by it will be destroyed in the reverse order, including the objects in arrays.
If at any point during the construction of the FlexibleClass an exception is thrown, all objects that were fully created will be destroyed in the reverse order of construction.
