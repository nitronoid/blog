
## Introduction

One of the pillars of modern C++ is that we should apply [RAII][1] wherever possible, to clarify ownership and shift the burden of managing object lifetimes from the programmer to the compiler.  
Unfortunately, as C++ developers we are often required to interface with legacy code, and libraries which offer a C-API.  
It's usually good practice, although tedious, to create [RAII][1] counterparts for types returned from these API's so that we can at least enforce good practice in our own code.  
Keeping these [RAII][1] types in sync with their less attractive cousins is usually a pain, we've violated the [DRY][2] principle and now need to remember to update both types where before we only needed to change one.
```cpp
struct DataLegacy
{
    int* indices;
    float* vertices;
    // New member was added!
    char* new_data;
};

struct DataRAII
{
    std::unique_ptr<int[]> indices;
    std::unique_ptr<float[]> vertices;
    // Missing a member for new_data!
};

DataRAII convert_to_RAII(DataLegacy&& legacy)
{
    // Losing the new_data here!
    return DataRAII{
        std::unique_ptr<int[]>{legacy.indices}, 
        std::unique_ptr<float[]>{legacy.vertices}
    };
}
```
It would be useful if we could enforce that the two types contained the same information, and could detect when they have become desynchronised.  

Enter, static-reflection.  

There have been a few proposals to add some form of static reflection to the language, however as of writing this post we have no official mechanisms to inspect the number of members in a struct.  
The compiler _does_ have this information though.  
In the rest of this post I will explain how we can brute force the compiler into telling us what we want to know.  

## The detection idiom

The detection idiom is a technique for determining whether a piece of code will fail to compile due to a logic error.  
We can use this to exploit details of certain types, forming some basic type traits.  
```cpp
// Base case, assume false
template <typename, typename>
static constexpr bool is_derefable_impl = false;

// If our inner statement *T{} compiles, then use true
template <typename T>
static constexpr bool is_derefable_impl<std::void_t<decltype(*T{})>, T> = true;

// Wrap up the template so the user doesn't need to pass "void"
template <typename T>
static constexpr bool is_dereferencable = is_derefable_impl<void, T>;
```
The trick is that the compiler will attempt to instantiate template specialisations before attempting to instantiate the un-specialised template.   If the statement was valid, the specialisation is used and we get a value of `true`, otherwise we get the unspecialised template with a value of `false`.  
[Try for yourself.](https://godbolt.org/z/KP9W9j)

## Counting struct members

Structs which allow aggregate initialisation do not require each of their members to receive an initialiser, within a braced-initialiser list. We are able to pass any number of arguments through the braced-init list up to the number of members in the struct, and no more, otherwise we will receive a compiler error.  
It is this detail that we shall exploit to count the number of members in a given struct.  
Consider the snippet below:  
```cpp
struct MyStruct
{
    int a;
    char const* b;
    float c;
};

// Compiles
MyStruct s0{0};
// Compiles
MyStruct s1{0, "Hello world"};
// Compiles
MyStruct s2{0, "Hello world", 42.f};
// Error, too many arguments to braced-init list
MyStruct s3{0, "Hello world", 42.f, 0};
```
We are going to attempt to construct an instance of the given type, using an ascending number of arguments, until construction fails.  
At the point of failure, we'll know that the number of members in the struct is equal to the number of arguments we used, minus one.  
First we can apply the detection idiom to write a trait for checking if an instance of type T can be instantiated with a provided type list.  
```cpp
// Base case assume false
template <typename, typename T, typename... Ts>
static constexpr bool is_list_initializable_impl = false;

// If the list init compiles, then true
template <typename T, typename... Ts>
static constexpr bool
    is_list_initializable_impl<std::void_t<decltype(T{std::declval<Ts>()...})>, T, Ts...> = true;

// Wrap up the template so the user doesn't need to pass "void"
template <typename T, typename... Ts>
static constexpr bool
    is_list_initializable = is_list_initializable_impl<void, T, Ts...>;
```
This lets us check things such as:
```cpp
struct MyStruct
{
    int a;
    char const* b;
    float c;
};
static_assert(is_list_initializable<MyStruct, int, char const*, float>);
static_assert(!is_list_initializable<MyStruct, int*>);
```
[Godbolt link.](https://godbolt.org/z/rYxnMs)

However we've had to manually specify the number of types, and get all of the types correct. Something a little more automatic would be useful.  
First, lets try to remove the need for specifying the exact types in the struct.  
```cpp
struct CvtToAny
{
    // Produce from any sequence of values
    template <typename... T> CvtToAny(T&&...) {}
    // Don't need to define this as it should never be instantiated/used
    template <typename T> constexpr operator T() noexcept;
};
```
We can define a new type which proclaims that it may convert to any other type, no need to worry about actually implementing the conversion as we'll only call this from an [unevaluated context](TODO).  
Using our new type, we can rewrite our above example:  
```cpp
// Passes
static_assert(is_list_initializable<MyStruct, CvtToAny>);
static_assert(is_list_initializable<MyStruct, CvtToAny, CvtToAny>);
static_assert(is_list_initializable<MyStruct, CvtToAny, CvtToAny, CvtToAny>);
// Fails, too many arguments to braced-init list
static_assert(is_list_initializable<MyStruct, CvtToAny, CvtToAny, CvtToAny, CvtToAny>);
```
[Godbolt link.](https://godbolt.org/z/z5Wb5P)

This is better, but since all of the types are now the same it would be nice to specify a number rather than list them out. To do this we can use a helper template:  
```cpp
// Base declaration to be specialised, required to extract indices
template <typename T, typename>
static constexpr bool is_list_initializable_n_impl = false;

// Helper to instantiate is_list_initializable_impl with N args
template <typename T, std::size_t... I>
static constexpr bool is_list_initializable_n_impl<T, std::index_sequence<I...>> =
    is_list_initializable<T, decltype(CvtToAny{I})...>;

// Determine whether a type T can be constructed from N arguments
template <typename T, std::size_t N>
static constexpr bool is_list_initializable_n =
    is_list_initializable_n_impl<T, std::make_index_sequence<N>>;
```
We've used a classic trick here and leveraged template specialisation to extract a pack of integers from the type `std::index_sequence`, this is the closest we can come to iteration with an index while in template land. This allows us to perform [pack expansion][3] and create an argument of type `CvtToAny` for each index in the pack.
And now our assertions look like this:
```cpp
static_assert(is_list_initializable_n<MyStruct, 1>);
static_assert(is_list_initializable_n<MyStruct, 2>);
static_assert(is_list_initializable_n<MyStruct, 3>);
static_assert(!is_list_initializable_n<MyStruct, 4>);
```
[Godbolt link.](https://godbolt.org/z/jdP14z)

We're getting much closer, we only need to find the magic number N now. It's time to employ some recursive templates:
```cpp
// Base template definition to be specialised
template <typename, std::size_t, bool>
static constexpr std::size_t member_count_impl = ~std::size_t{0};

/*
 * Case hit when construction failed, subtract one since we eagerly increment
 * and another because this is the number of arguments that produces an error, 
 * aka the number of members +1
 */
template <typename T, std::size_t N>
static constexpr std::size_t member_count_impl<T, N, false> = N - 2;

// Recurse while construction succeeds
template <typename T, std::size_t N>
static constexpr std::size_t member_count_impl<T, N, true> =
  member_count_impl<T, N + 1, is_list_initializable_n<T, N>>;

// Determine the number of members within type T
template <typename T>
static constexpr std::size_t member_count = member_count_impl<T, 0, true>;
```
What we're doing is instantiating `member_count_impl` using `is_list_initializable_n` as our recursion guard. As soon as the list initialisation fails, our `false` specialisation is instantiated and recursion is terminated.  
Lets see how this looks with some more assertions:  
```cpp
struct MyStruct
{
    int a;
    char const* b;
    float c;
};
static_assert(member_count<MyStruct> == 3);
static_assert(member_count<MyStruct> != 4);
```
[Godbolt link.](https://godbolt.org/z/K4Pf9K)

And there we have it, a method for counting the number of members in any struct which supports braced-list initialisation.  


## Nested Unions and structs
Union and Struct members are slightly trickier to handle since they require their own set of nested braces to be initialised. This sounds simple since extra braces for non-struct/union members are ignored, however not all compilers agree. Since GCC 9, a consensus has been reached and the extra braces are ignored by all the major compilers.  
```cpp
struct Outer
{
  float x;
  union
  {
    int a;
    int b;
  };
};
// Error, missing braces to initialise the union member
Outer o{CvtToAny{}, CvtToAny{}};
// Error with GCC < 9, redundant braces around first argument
Outer o{{CvtToAny{}}, {CvtToAny{}}};
```
Fixing our implementation to support this is trivial, we can add a set of braces around each list parameter, since extra braces around parameters which don't require them are simply ignored.  
```cpp
template <typename, typename T, typename... Ts>
static constexpr bool is_list_initializable_impl = false;

// Extra braces here -----------------------------------------------------
//                                                    V                  V
template <typename T, typename... Ts>
static constexpr bool
    is_list_initializable_impl<std::void_t<decltype(T{{std::declval<Ts>()}...})>, T, Ts...> = true;
```

## C++20 concepts
C++20 comes with the addition of concepts to the language. We can leverage concepts to replace our detection idiom with a simpler bit of code.  
Unfortunately we still need the index sequence trickery to create a list of N arguments, but our first trait is simplified.  
```cpp
// Use a requires expression to ensure that the type T can be created using a list containing Ts...
template<typename T, typename... Ts>
concept ListInitializable = requires {{ T{{std::declval<Ts>()}...} };};

// Base declaration to be specialised, required to extract indices
template <typename T, typename> constexpr bool is_list_initializable_n_impl = false;

// Helper to instantiate is_list_initializable_impl with N args
template <typename T, std::size_t... I>
constexpr bool is_list_initializable_n_impl<T, std::index_sequence<I...>>
  = ListInitializable<T, decltype(CvtToAny{I})...>;

// Final concept
template <typename T, std::size_t N>
concept ListInitializableN
    = is_list_initializable_n_impl<T, std::make_index_sequence<N>>;
```
[Godbolt link.](https://godbolt.org/z/cThPze)

## Exponential searching
We've implemented a linear search algorithm, which when coupled with recursive templates, may result in a lot of template instantiations.
It's possible to instead use an [exponential search][4], a `O(log N)` algorithm, to incur less template instantiations.
We need to use the exponential search rather than a binary search since our range is unbounded, however this will probably lead to better results anyway since we can expect that most structs have few members.
We would need to instead look at pairs of list initialisations to find a couple where the first list of length `N` succeeds and the second of length `N + 1` fails.

### Footnotes
* [Find all of the code in this post here.](https://github.com/nitronoid/member_count)
* [RAII][1] "Resource Acquisition Is Initialisation"
* [DRY][2] "Don't repeat yourself"
[1]: <https://en.cppreference.com/w/cpp/language/raii#:~:text=Resource%20Acquisition%20Is%20Initialization%20or,in%20limited%20supply)%20to%20the> (RAII)
[2]: <https://en.wikipedia.org/wiki/Don%27t_repeat_yourself> (DRY)
[3]: <https://en.cppreference.com/w/cpp/language/parameter_pack> (Pack expansion)
[4]: <https://en.wikipedia.org/wiki/Exponential_search> (Exponential search)
