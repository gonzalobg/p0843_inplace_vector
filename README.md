`inplace_vector`
===

> A dynamically-resizable vector with fixed capacity and embedded storage

**Document number**: P0843R10.
**Date**: 2024-02-12.
**Authors**: Gonzalo Brito Gadeschi, Timur Doumler <papers \_at\_ timur.audio>, Nevin Liber, David Sankel <dsankel \_at\_ adobe.com>.
**Reply to**: Gonzalo Brito Gadeschi <gonzalob \_at\_ nvidia.com>.
**Audience**: LWG.

<style>
ins {
    color:green;
    background-color:yellow;
    text-decoration:underline;
}
del {
    color:red;
    background-color:yellow;
    text-decoration:line-through;
}
bdi {
    color:black;
    background-color:lightblue;
    text-decoration:underline;
}
.markdown-body {
    max-width: 900px;
    text-align: justify;
}
</style>

<big>Table of Contents</big>

[toc]

**<big>Changelog</big>**

* **Revision 10** 
  * Extend `constexpr` from "trivial" types to "literal" types. 
  * Remove `unchecked_append_range`: adds very little value (one branch amortized over all inserted elements).
  * Update remarks of mutating elements to specify that if an exception occurs while inserting elements, the succesfully inserted elements are kept.
  * Discussion of execption safety guarantees for mutating operations and outcome from LEWG discussion.
  * Should not be allocator away.
  * Should throw `bad_alloc` on exceeding capacity.
  * Should be in a separate header.
  * Added fallible `append_range` APIs.
  * Move iterator erase methods from \[vector.erasure\] to \[vector.modifiers\].
  * Updated some EDITORIAL notes.
  * Fixed typo in \[vector.modifiers\], the `insert_range` method was incorrectly named `insert`.
  * Add accidentally missing `append_range` to \[vector.modifiers\].
  * Removed unnecessary _Complexity_ clauses from `resize` methods.
* **Revision 9** Varna 2023
  * All preconditions on `sz < capacity` are now a "Throws `bad_alloc`" with the exception of the "`unchecked_`" family of functions.
  * The "`try_`" family of insertion functions do not consume the input rvalue references if the container is full.
  * Container move / copy constructor are trivial if `T` is trivial  move / copy constructible.
  * Swap member function is now noexcept if `N == 0` or value type has nothrow move constructors.
  * Made complexity of resize linear. 
  * Fixed out-of-bounds math in wording (less than equal to vs less).
  * Fixed constraints on all the "`emplace` family" of functions. 
  * Fixed constraints of unary constructor taking a size to require default insertability instead of copy insertability.
  * Fixed missing angle brackets on `<inplace_vector>` header and listed headers alphabetically.
  * Cleanup: removed duplicates of preconditions that are covered in the sequence container requirements.
  * Cleanup: removed unnecessary specification of member swap and specialized algorithms.
  * Styling: use `class` instead of `typename` in template heads, replace `value_type` with `T` in wording, `bad_alloc` in code font, etc. 
* **Revision 8** Varna 2023
    * Added LEWG poll showing consensus for `<inplace_vector>` header.
    * Add feature test macro
    * Add `try_push_back` and `unchecked_push_back` to wording.
    * Add `at` to `inplace_vector` class synopsis.
    * Add range construction and assignment.
    * Add missing `reserve` method that throws `bad_alloc` if `capacity()` is exceeded.
    * Add missing `shrink_to_fit` method that has no effects. 
    * Add missing `insert_range`. 
    * Add wording for move constructor semantics (trivial if `T` is trivial).
    * Add wording for destructor semantics (trivial if `T` is trivial). 
    * Remove deduction guidelines since cannot deduce `capacity()` meaningfully.
    * Add to containers.sequences.general.
    * Add to sequence containers table.
    * Add to iterator.range.
    * Add to diff.cpp03.library.
    * Add poll result confirming unchecked_push_back.
    * Add erasure.
    * Add poll result confirming the overall design.
    * Review synopsis/wording for other missing functions.
    * Update `operator==` to `operator<=>` using hidden friends for them.
    * Made `<inplace_vector>` not freestanding (this will be handled in a separate paper).
* **Revision 7** Varna 2023
    * Rename `static_vector` to `inplace_vector` throughout.
    * Update `try_push_back` APIs to return `T*` with rationale.
    * Update `push_back` to throw `std::bad_alloc` with rationale .
    * Trivially-copyable if `value_type` is trivially-copyable.
    * Request LEWG poll regarding `<vector>` or `<inplace_vector>` header.
    * Make `push_back` return a reference
* **Revision 6**: for Varna 2023 following Kona's 2022 guidance
    * Updated push_back semantics to follow std::vector (note about exception to throw).
    * Added `try_push_back` returning an `optional`
    * Added `push_back_unchecked`: excedding capacity exhibits undefined behavior.
    * Added note about naming.
* **Revision 5**:
    * Update contact wording and contact data.
    * Removed naming discussion, since it was resolved (last available in [P0843r4](https://wg21.link.p0843r4)).
    * Removed future extensions discussion (last available in [P0843r4](https://wg21.link.p0843r4)).
    * Addressed LEWG feedback regarding move-semantics and exception-safety.
* **Revision 4**:
    * LEWG suggested that push_back should be UB when the capacity is exceeded
    * LEWG suggested that this should be a free-standing header
* **Revision 3**:
    * Include LWG design questions for LEWG.
    * Incorporates LWG feedback.
* **Revision 2**
    * Replace the placeholder name `fixed_capacity_vector` with `static_vector`
    * Remove at checked element access member function.
    * Add changelog section.
* **Revision 1**
    * Minor style changes and bugfixes.

# Introduction

This paper proposes `inplace_vector`, a dynamically-resizable array with capacity fixed at compile time and contiguous inplace storage, that is, the array elements are stored within the vector object itself. Its API closely resembles `std::vector<T, A>`, making it easy to teach and learn, and the _inplace storage_ guarantee makes it useful in environments in which dynamic memory allocations are undesired.

This container is widely-used in the standard practice of C++, with prior art in, e.g., [`boost::static_vector<T, Capacity>` [1]][boost_static_vector] or the [EASTL [2]][eastl], and therefore we believe it will be very useful to expose it as part of the C++ standard library, which will enable it to be used as a vocabulary type.


# Motivation and Scope

The `inplace_vector` container is useful when:

* memory allocation is not possible, e.g., embedded environments without a free store, where only automatic storage and static memory are available;
* memory allocation imposes an unacceptable performance penalty, e.g., in terms of latency;
* allocation of objects with complex lifetimes in the _static_-memory segment is required;
* the storage location of the `inplace_vector` elements is required to be within  the `inplace_vector` object itself, e.g., for serialization purposes (e.g. via `memcpy`);
* `std::array` is not an option, e.g., if non-default constructible objects must be stored; or
* a dynamically-resizable array is needed during constant evaluation.

# Existing practice

Three widely used implementations of `inplace_vector` are available: [Boost.Container [1]][boost_static_vector], [EASTL [2]][eastl], and [Folly [3]][folly]. `Boost.Container` implements `inplace_vector` as a standalone type with its own guarantees. [EASTL][eastl] and [Folly][folly] implement it via an extra template parameter in their `small_vector` types.

Custom allocators like [Howard Hinnant's `stack_alloc` [4]][stack_alloc] emulate `inplace_vector` on top of `std::vector`, but as discussed in the next sections, this emulation is not great.

Other prior art includes the following.

* [P0494R0: `contiguous_container` proposal [5]][contiguous_container]: proposes a `Storage` concept.
* [P0597R0: `std::constexpr_vector<T>` [6]][constexpr_vector_1]: proposes a vector that can only be used in constexpr contexts.

A reference implementation of this proposal is available [here (godbolt)][reference_implementation].

[reference_implementation]: https://godbolt.org/z/5P78aG5xE

# Design

The design described below was approved at [LEWG Varna '23](todo):

* **POLL:** The provided signatures and semantics that D08437R7 provides for push_back, emplace_back, try_push_back, try_emplace_back, and the unchecked versions are acceptable.

|Strongly Favor|Weakly Favor|Neutral |Weakly Against |Strongly Against|
|-|-|-|-|-|
|9|7|0|0|0|

* **POLL**: We approve the design of D0843R7 (inplace_vector) with the changes already polled.

|Strongly Favor|Weakly Favor|Neutral |Weakly Against |Strongly Against|
|-|-|-|-|-|
|11|5|0|0|0|

## Standalone or a special case another type?

The [EASTL][eastl] [2] and [Folly][folly] [3] special case `small_vector`, e.g., using a fourth template parameter, to make it become an `inplace_vector`. [P0639R0: _Changing attack vector of the `constexpr_vector`_ [7]][constexpr_vector_2] proposes improving the `Allocator` concepts to allow implementing `inplace_vector` as a special case of `vector` with a custom allocator. Both approaches produce specializations of `small_vector` or `vector` whose methods differ subtly in terms of effects, exception safety, iterator invalidation, and complexity guarantees.

This proposal closely follows [`boost::container::static_vector<T,Capacity>` [1]][boost_static_vector] and proposes `inplace_vector` as a standalone type.

Where possible, this proposal defines the semantics of `inplace_vector` to match `vector`. Providing the same programming model makes this type easier to teach and use, and makes it easy to "just change" one type in a program to, e.g., perform a performance experiment without accidentally introducing undefined behavior.

## Layout

`inplace_vector` models `ContiguousContainer`. Its elements are stored and properly aligned within the `inplace_vector` object itself. If the `Capacity` is zero the container has zero size:

```cpp
static_assert(is_empty_v<inplace_vector<T, 0>>); // for all T
```

The offset of the first element within `inplace_vector` is unspecified, and `T`s are not allowed to overlap.

The layout differs from `vector`, since `inplace_vector` does not store the `capacity` field (it's known from the template parameter).

If `T` is trivially-copyable or `N == 0`, then `inplace_vector<T, N>` is also trivially copyable to support high-performance computing (HPC) use cases, such as the following.

* Copying between host and accelerator memory spaces.  Examples of accelerators include Graphics Processing Units (GPUs).
* Serialization and deserialization for distributed-memory parallel communication, e.g., sending a vector via the `MPI_Send` function from the Message Passing Interface (MPI).

```cpp
// for all C:
static_assert(!is_trivially_copyable_v<T> || is_trivially_copyable_v<inplace_vector<T, C>> || N == 0);
```

## Move semantics

A moved-from `inplace_vector` is left in a _valid but unspecified state_ (option 3 below) unless `T` is trivially-copyable, in which case the size of the `inplace_vector` does not change (`array` semantics, option 2 below). That is:

```cpp=
inplace_vector a(10);
inplace_vector b(std::move(a));
assert(a.size() == 10); // MAY FAIL
```

moves `a`'s elements element-wise into `b`, and afterwards the size of the moved-from `inplace_vector` may have changed.

This prevents code from relying on the size staying the same (and therefore being incompatible with changing an `inplace_vector` type back to `vector`) without incuring the cost of having to clear the `inplace_vector`.

When `T` is trivially-copyable, `array` semantics are used to provide trivial move operations.
    
This is different from [LEWG Kona '22 Polls](https://github.com/cplusplus/papers/issues/114#issuecomment-1312015872) (22 in person + 8 remote) and we'd like to poll on these semantics again:
    
* **POLL**: Moving a static_vector should empty it (vector semantics).

  | Strongly Favor |	Weakly Favor |	Neutral	| Weakly Against	| Strongly Against| 
  |-|-|-|-|-|
  |9	| 10 | 	4 | 	2 | 	2 | 
    
* **POLL**: Moving a static_vector should leave it in a valid but unspecified state.

  | Strongly Favor |	Weakly Favor |	Neutral	| Weakly Against	| Strongly Against| 
  |-|-|-|-|-|
  |6	| 9 | 	1 | 	5 | 	6 | 
    

**Alternatives**:

1. `vector` semantics: guarantees that `inplace_vector` is left empty (this happens with move assignment when using `std::allocator<T>` and always with move construction).
    * Pro: same programming model as `vector`.
    * Pro: increases safety by requiring users to re-initialize vector elements.
    * Con: clearing an `inplace_vector` is not free.
    * Con: `inplace_vector<T, N>` can no longer be made trivially copyable for a trivially copyable `T`, as the move operations can no longer be trivial.
2. `array` semantics: guarantees that `size()` of `inplace_vector` does not change, and that elements are left in their moved-from state.
    * Pro: no additional run-time cost incurred.
    * Con: different programming model than `vector`.
3. "valid but unspecified state"
    * Con: different programming model than `vector` and `array`, requires calling `size()`
    * Pro: code calling `size()` is correct for both `vector` and `inplace_vector`, enabling changing the type back and forth.

## Exception Safety

When using the `inplace_vector` APIs, the following types of failures are expected:

* May throw:
  1. The `value_type`'s constructors/assignment/destructors/swap (depends on `noexcept`),
  2. Mutating operations exceeding the capacity (`push_back`, `insert`, `
    `, `inplace_vector(value_type, size)`, `inplace_vector(begin, end)`...), and
  3. Out-of-bounds checked access: `at`.

* Pre-condition violation:
  1. Out-of-bounds unchecked access: `front`/`back`/`pop_back` when empty, `operator[]`.
 

### Exception Safety guarantees of Mutating Operations

When an `inplace_vector` API throws an exception, 
* _Basic Exception Guarantee_ requires the API to leave the `inplace_vector` in a valid state.
* _Strong Exception Guarantee_ requires the API to roll back the `inplace_vector` state to that of before the API was called, e.g., removing previously inserted elements, and loosing data when inserting from input iterators or ranges.

The following alternative were considered:
1. Same guarantees as their counter-part `vector` APIs.
1. Always provide the _Basic Guarantee_ independent on the concepts implemented by the iterators/ranges: always insert up to the capacity, then throw.
1. Provide different exception safety guarantees depending on the concepts modeled by the iterators/ranges API arguments:
    - `sized_range`,  `random_access_iterator`, or `LegacyRandomAccessIterator`: _Strong guarantee_, i.e., if the capacity would be exceeded, the API throws without attempting to insert any elements. This performs well and the caller looses no data.
    - Otherwise: _Basic guarantee_, i.e., elements are inserted up to the capacity, and are not removed before throwing. This performs well and the caller only looses data, e.g., stashed in discarded input iterators.

We propose to, unless stated otherwise, `inplace_vector` APIs should provide the same exception safety guarantees as their counter-part `vector` APIs. 

### Exception thrown by mutating operations exceeding capacity


We propose that mutating operations that exceed the capacity throw `bad_alloc`, to make it safer for applications handling out of memory errors to introduce `inplace_vector` as a performance optimization by replacing `vector`.

LEWG revisited the rationale below and decided to keep throwing `bad_alloc` in the 2024-01-30 telecon.

**Alternatives**:

1. Throw `bad_alloc`: `inplace_vector` requests storage from "allocator embedded within the `inplace_vector`", which fails to allocate, and therefore throws `bad_alloc` (e.g. like `vector` and `pmr` "stack allocator").
    * **Pros**: handling `bad_alloc` is more common than other exceptions when attempting to handle failure to insert due to "out-of-memory".
3. Throw `length_error`: insertion exceeds `max_size` and therefore throws `length_error`
    * **Pros**: container requirements already imply that this exception may be thrown.
    * **Cons**: handling `length_error` is rare since it is usually very high.
5. Throw "some other exception" when the `inplace_vector` is out-of-memory:
    * **Pros**: to be determined.
    * **Cons**: different programming model as `vector`.
6. Abort the process
    * **Pros**: portability to embedded platforms without exception support
    * **Cons**: different programming model than `vector`
7. Precondition violation
    * **Cons**: different proramming model than `vector`, users responsible for checking before modifying vector size, etc.


## Fallible APIs

We add the following new fallible APIs which, when the vector size equal its capacity, return `nullptr` (and do not throw `bad_alloc`) without moving from the inputs, enabling them to be re-used:

```cpp
constexpr T* inplace_vector<T, C>::try_push_back(const T& value);
constexpr T* inplace_vector<T, C>::try_push_back(T&& value); 

template<class... Args>
  constexpr T* try_emplace_back(Args&&... args);

template< container-compatible-range<T> R>
  constexpr ranges::iterator_t<R> try_append_range(R&& rg);
```

The `try_append_range` API always tries to insert all `rg` range elements up to either the vector capacity or the range `rg` is exhausted. It returns an iterator to the first non-inserted element of `rg` or the end iterator of `rg` if the range was exhausted. It intentionally provides the Basic Exception Safety guarantee, i.e., if inserting an element throws, previously succesfully inserted elements are preserved in the vector (i.e. not lost).

These APIs may be used as follows:

```cpp=
T value = T();
if (!v.try_push_back(value)) {
    std::cerr << "Failed to insert " << value << std::endl; // value not moved-from
    std::terminate();
}

auto il = {1, 2, 3};
if (v.try_append_range(il) != end(il)) {
    // The vector capacity was exhausted
    std::terminate();
}
```

## Fallible Unchecked APIs

We add the following new fallible unchecked APIs for which exceeding the capacity is a precondition violation:

```cpp
constexpr T& inplace_vector<T, C>::unchecked_push_back(const T& value);
constexpr T& inplace_vector<T, C>::unchecked_push_back(T&& value);

template<class... Args>
  constexpr T& unchecked_emplace_back(Args&&... args);

template< container-compatible-range<T> R>
  constexpr ranges::borrowed_iterator_t<R> unchecked_append_range(R&& rg);
```

The `append_range` API was requested during LWG review in December 2023.

These APIs were requested in [LEWG Kona '22](https://github.com/cplusplus/papers/issues/114#issuecomment-1312015872) (22 in person + 8 remote):
    
* **POLL**: If static_vector has unchecked operations (e.g. `push_back_unchecked`), it is okay for checked operations (e.g. `push_back`) to throw when they run out of space.

  | Strongly Favor |	Weakly Favor |	Neutral	| Weakly Against	| Strongly Against| 
  |-|-|-|-|-|
  |14	| 4 | 	2 | 	4 | 	1 | 

This was confirmed at [LEWG Varna '23](todo) after a discussion on safety:
* **POLL**: D0843R7 should remove the unchecked versions of push_back and emplace_back

|Strongly Favor|Weakly Favor|Neutral |Weakly Against |Strongly Against|
  |-|-|-|-|-|
|1|5|3|3|7|


The name `unchecked_push_back` was polled in [LEWG Varna '23](todo):

* **POLL**: (vote for all the options you find acceptable, vote as many times as you like) Feature naming

  | Feature name | Votes |
  |-|-|
  |push_back_unchecked|11|
  |unchecked_push_back|16|
  |unsafe_push_back|9|
  |push_back_unsafe|6|


The potential impact of the three APIs on code size and performance is shown [here](https://clang.godbolt.org/z/MbG17q8x1), where the main difference between `try_push_back` and `unchecked_push_back` is the presence of an extra branch in `try_push_back`.

## Allocator awareness

We believe that right now, making `inplace_vector` allocator-aware does not outweigh its complexity and design cost. We can always provide a way to support that in the future.

Options:
- `inplace_vector` is allocator-aware if its `value_type` is allocator-aware.
- factoring an allocator-aware `inplace_vector` into a separate `basic_allocator` class.
- no support for now (not worth delaying further)

## Iterator invalidation

`inplace_vector` iterator invalidation guarantees differ from `std::vector`:

- moving a `inplace_vector` invalidates all iterators, and
- swapping two `inplace_vector`s invalidates all iterators.

`inplace_vector` APIs that potentially invalidate iterators are: `resize(n)`, `resize(n, v)`, `pop_back`, `erase`, and `swap`.

## Freestanding

Many`inplace_vector` APIs are not available in freestanding because fallible insertion APIs (constructors, push back, insert, ...) may throw. 

The infallible `try_` APIs do not throw and are available in freestanding. They only cover a subset of the functionality available through fallible APIs. This is intentional. Adding more infallible APIs to `inplace_vector` and potentially other containers is left as future work.
  
We'd need to add it to: [\[library.requirements.organization.compliance\]]

[\[library.requirements.organization.compliance\]]: http://eel.is/c++draft/compliance

When we fix this we'd need to add  `<inplace_vector>` to [\[tab:headers.cpp.fs\]]:
    
| | Subclause | Headers |
|-|-|-|
| <ins>[\[containers\]]</ins> | <ins>containers</ins> | <ins>`<inplace_vector>`</ins> |
    
[\[tab:headers.cpp.fs\]]: http://eel.is/c++draft/tab:headers.cpp.fs

    
## Same or Separate header

We propose that this container goes into its own header `<inplace_vector>` rather than in header `<vector>`, because it is a sufficiently different container.

LWG asked for `inplace_vector` to be part of the `<vector>` header. [LEWG Varna '23]() took the following poll:

* **POLL:** D0843R7 should provide `inplace_vector` in `<vector>` rather than the proposal’s decision on `<inplace_vector>`

  | Strongly Favor |	Weakly Favor |	Neutral	| Weakly Against	| Strongly Against| 
  |-|-|-|-|-|
  |0	| 0 | 	1 | 	12 | 	5 | 

That is, consensus against change.

## Return type of push_back

In C++20, both `push_back` and `emplace_back` were slated to return a `reference` (they used to both return `void`).  Even with plenary approval, changing `push_back` turned out to be an ABI break that was backed out, leaving the situation where `emplace_back` returns a `reference` but `push_back` is still `void`.  This ABI issue doesn't apply to new types.  Should `push_back` return a `reference` to be consistent with `emplace_back`, or should it be consistent with older containers?

Request LEWG to poll on that.

## `reserve` and `shrink_to_fit` APIs
    
`shrink_to_fit` requests `vector` to decrease its `capacity`, but this request may be ignored. `inplace_vector` may implement it as a nop (and it may be `noexcept`).
    
`reserve(n)` requests the `vector` to potentially increase its `capacity`, failing if the request can't be satisfied. `inplace_vector` may implement it as a nop if `n <= capacity()`, throwing `bad_alloc` otherwise.
       
These APIs make it easier and safe for programs to be "more" parametric over "vector-like" containers (`vector`, `small_vector`, `inplace_vector`), but since they do not do anything useful for `inplace_vector`, we may want to fail to compile instead.

## Deduction guides

Unlike the other containers, `inplace_vector` does not have any deduction guides because there is no case in which it would be possible to deduce the second template argument, the capacity, from the initializer.

    
## Summary of semantic differences with `vector`

| **Aspect** | `vector` | `inplace_vector` |
|------------|----------|------------------|
| Capacity | Indefinite | `N`
| Move and swap | O(1), no iterators invalidated | `array` semantics: O(size), invalidates all iterators |
| Moved from | left empty (this happens with move assignment when using std::allocator<T> and always with move construction) | valid but unspecified state except if `T` is trivially-copyable, in which case `array` semantics |
| Default construction and destruction of trivial types | O(1) | O(capacity)
| Is empty when zero capacity? | No | Yes |
| Trivially-copyable if `is_trivially_copyable_v<T>`? | No | Yes | 

## Name

The class template name was confirmed at [LEWG Varna '23](todo):

* **POLL**:  Feature naming

|Options|Votes|
|-|-|
|static_vector|4|
|inplace_vector|14|
|fixed_capacity_vector|5|

# Technical specification

<bdi>**EDITORIAL:** This enhancement is a pure header-only addition to the C++ standard library as the `<inplace_vector>` header. It belongs in the "Sequence containers" ([\[sequences\]]) part of the "Containers library" ([\[containers\]]) as "Class template `inplace_vector`".</bdi>

[\[sequences\]]: http://eel.is/c++draft/sequences
[\[containers\]]: http://eel.is/c++draft/containers

## [\[library.requirements.organization.headers\]]

[\[library.requirements.organization.headers\]]: http://eel.is/c++draft/headers
    
Add <ins>`<inplace_vector>`</ins> to [\[tab:headers.cpp\]].
    
Add  `<inplace_vector>` to [\[tab:headers.cpp.fs\]]:
    
| | Subclause | Headers |
|-|-|-|
| <ins>[\[containers\]]</ins> | <ins>containers</ins> | <ins>`<inplace_vector>`</ins> |
    
[\[tab:headers.cpp\]]: http://eel.is/c++draft/tab:headers.cpp

## [\[iterator.range\]] Range access

Modify:

[1](https://eel.is/c++draft/iterator.range#1) In addition to being available via inclusion of the `<iterator>` header, the function templates in [\[iterator.range\]] are available when any of the following headers are included: `<array>`, `<deque>`, `<forward_list>`, <ins>`<inplace_vector>`, </ins>`<list>`, `<map>`, `<regex>`, `<set>`, `<span>`, `<string>`, `<string_view>`, `<unordered_map>`, `<unordered_set>`, and `<vector>`.

[\[iterator.range\]]: https://eel.is/c++draft/iterator.range
    
## [\[container.alloc.reqmts\]]
 
Modify:
 
[1](http://eel.is/c++draft/container.alloc.reqmts#1) All of the containers defined in [\[containers\]] and in [\[basic.string\]] except `array`<ins> and `inplace_vector`</ins> meet the additional requirements of an allocator-aware container, as described below.

[\[container.alloc.reqmts\]]: http://eel.is/c++draft/container.alloc.reqmts
[\[basic.string\]]: http://eel.is/c++draft/basic.string 
    
## [\[allocator.requirements.general\]]
  
[1](https://eel.is/c++draft/allocator.requirements.general#1) The library describes a standard set of requirements for _allocators_, which are class-type objects that encapsulate the information about an allocation model. This information includes the knowledge of pointer types, the type of their difference, the type of the size of objects in this allocation model, as well as the memory allocation and deallocation primitives for it. All of the string types, containers (except `array`<ins> and `inplace_vector`</ins>), string buffers and string streams ([\[input.output\]]), and match_results are parameterized in terms of allocators.

[\[allocator.requirements.general\]]: https://eel.is/c++draft/allocator.requirements.general
    
[\[input.output\]]: https://eel.is/c++draft/input.output
    
## [\[containers.general\]]
    
Modify [\[tab:containers.summary\]]:
    
| | Subclause | Headers |
|-|-|-|
| [\[sequences\]] | Sequence containers | <array>, <deque>, <forward_list><ins>`, <inplace_vector>`</ins>, <list>, <vector> |

[\[tab:containers.summary\]]: http://eel.is/c++draft/tab:containers.summary
[\[containers.general\]]: http://eel.is/c++draft/containers.general    
    
## [\[container.reqmts\]] General container requirements

[\[container.reqmts\]]: http://eel.is/c++draft/container.reqmts

1. A type `X` meets the container requirements if the following types, statements, and expressions are well-formed and have the specified semantics.

```cpp
typename X::value_type
```
* Result: `T`
* Preconditions: `T` is `Cpp17Erasable` from `X` (see [container.alloc.reqmts], below).

```cpp
typename X::reference
```
* Result: `T&`

```cpp
typename X::const_reference
```
* Result: `const T&`

```cpp
typename X::iterator
```
* Result: A type that meets the forward iterator requirements ([forward.iterators]) with value type `T`. The type `X::iterator` is convertible to `X::const_iterator`.

```cpp
typename X::const_iterator
```
* Result: A type that meets the requirements of a constant iterator and those of a forward iterator with value type `T`.

```cpp
typename X::difference_type
```
* Result: A signed integer type, identical to the difference type of `X::iterator` and `X::const_iterator`.

```cpp
typename X::size_type
```
* Result: An unsigned integer type that can represent any non-negative value of `X::difference_type`.

```cpp
X u;
X u = X();
```
* Postconditions: `u.empty()`
* Complexity: Constant.

```cpp
X u(a);
X u = a;
```
* Preconditions: `T` is `Cpp17CopyInsertable` into `X` (see below).
* Postconditions: `u == a`
* Complexity: Linear.

```cpp
X u(rv);
X u = rv;
```
* Postconditions: `u` is equal to the value that `rv` had before this construction.
* Complexity: Linear for array<ins> and `inplace_vector`</ins> and constant for all other standard containers.

```cpp
a = rv
```
* Result: `X&`.
* Effects: All existing elements of `a` are either move assigned to or destroyed.
* Postconditions: If `a` and `rv` do not refer to the same object, `a` is equal to the value that `rv` had before this assignment.
* Complexity: Linear.

```cpp
a.~X()
```
* Result: `void`
* Effects: Destroys every element of `a`; any memory obtained is deallocated.
* Complexity: Linear.

```cpp
a.begin()
```
* Result: `iterator`; `const_iterator` for constant `a`.
* Returns: An iterator referring to the first element in the container.
* Complexity: Constant.

```cpp
a.end()
```
* Result: `iterator`; `const_iterator` for constant `a`.
* Returns: An iterator which is the past-the-end value for the container.
* Complexity: Constant.

```cpp
a.cbegin()
```
* Result: `const_iterator`.
* Returns: `const_cast<X const&>(a).begin()`
* Complexity: Constant.

```cpp
a.cend()
```
* Result: `const_iterator`.
* Returns: `const_cast<X const&>(a).end()`
* Complexity: Constant.

```cpp
i <=> j
```
* Result: `strong_ordering`.
* Constraints: `X::iterator` meets the random access iterator requirements.
* Complexity: Constant.

```cpp
a == b
```
* Preconditions: `T` meets the `Cpp17EqualityComparable` requirements.
* Result: Convertible to `bool`.
* Returns: `equal(a.begin(), a.end(), b.begin(), b.end())` [Note 1: The algorithm `equal` is defined in [alg.equal]. — end note]
* Complexity: Constant if `a.size() != b.size()`, linear otherwise.
* Remarks: `==` is an equivalence relation.

```cpp
a != b
```
* Effects: Equivalent to `!(a == b)`.
```cpp
a.swap(b)
```
* Result: `void`
* Effects: Exchanges the contents of `a` and `b`.
* Complexity: Linear for array<ins> and `inplace_vector`,</ins> and constant for all other standard containers.

```cpp
swap(a, b)
```
* Effects: Equivalent to `a.swap(b)`.

```cpp
r = a
```
* Result: `X&`.
* Postconditions: `r == a`.
* Complexity: Linear.

```cpp
a.size()
```
* Result: `size_type`.
* Returns: `distance(a.begin(), a.end())`, i.e. the number of elements in the container.
* Complexity: Constant.
* Remarks: The number of elements is defined by the rules of constructors, inserts, and erases.

```cpp
a.max_size()
```
* Result: `size_type`.
* Returns: `distance(begin(), end())` for the largest possible container.
* Complexity: Constant.

```cpp
a.empty()
```
* Result: Convertible to `bool`.
* Returns: `a.begin() == a.end()`
* Complexity: Constant.
* Remarks: If the container is empty, then `a.empty()` is true.

1. In the expressions
```cpp
i == j
i != j
i < j
i <= j
i >= j
i > j
i <=> j
i - j
```
where `i` and `j` denote objects of a container's iterator type, either or both may be replaced by an object of the container's `const_iterator` type referring to the same element with no change in semantics.

Unless otherwise specified, all containers defined in this Clause obtain memory using an allocator (see [\[allocator.requirements\]]).
    
[\[allocator.requirements\]]: https://eel.is/c++draft/allocator.requirements

[Note 2: In particular, containers and iterators do not store references to allocated elements other than through the allocator's pointer type, i.e., as objects of type `P` or `pointer_traits<P>::template rebind<unspecified>`, where `P` is `allocator_traits<allocator_type>::pointer`. — end note]

Copy constructors for these container types obtain an allocator by calling `allocator_traits<allocator_type>::select_on_container_copy_construction` on the allocator belonging to the container being copied. Move constructors obtain an allocator by move construction from the allocator belonging to the container being moved. Such move construction of the allocator shall not exit via an exception. All other constructors for these container types take a `const allocator_type&` argument.

[Note 3: If an invocation of a constructor uses the default value of an optional allocator argument, then the allocator type must support value-initialization. — end note]

A copy of this allocator is used for any memory allocation and element construction performed, by these constructors and by all member functions, during the lifetime of each container object or until the allocator is replaced. The allocator may be replaced only via assignment or `swap()`. Allocator replacement is performed by copy assignment, move assignment, or swapping of the allocator only if
  1. `allocator_traits<allocator_type>::propagate_on_container_copy_assignment::value`,
  1. `allocator_traits<allocator_type>::propagate_on_container_move_assignment::value`, or
  1. `allocator_traits<allocator_type>::propagate_on_container_swap::value`
is `true` within the implementation of the corresponding container operation. In all container types defined in this Clause, the member `get_allocator()` returns a copy of the allocator used to construct the container or, if that allocator has been replaced, a copy of the most recent replacement.

The expression `a.swap(b)`, for containers `a` and `b` of a standard container type other than `array`<ins> and `inplace_vector`</ins>, shall exchange the values of `a` and `b` without invoking any move, copy, or swap operations on the individual container elements. Lvalues of any Compare, Pred, or Hash types belonging to `a` and `b` shall be swappable and shall be exchanged by calling swap as described in [\[swappable.requirements\]]. If `allocator_traits<allocator_type>::propagate_on_container_swap::value` is `true`, then lvalues of type `allocator_type` shall be swappable and the allocators of `a` and `b` shall also be exchanged by calling `swap` as described in [\[swappable.requirements\]]. Otherwise, the allocators shall not be swapped, and the behavior is undefined unless `a.get_allocator() == b.get_allocator()`. Every iterator referring to an element in one container before the swap shall refer to the same element in the other container after the swap. It is unspecified whether an iterator with value `a.end()` before the swap will have value `b.end()` after the swap.

Unless otherwise specified (see [associative.reqmts.except], [unord.req.except], [deque.modifiers],<ins> [inplace.vector.modifiers] </ins>and [vector.modifiers]) all container types defined in this Clause meet the following additional requirements:
1. If an exception is thrown by an insert() or emplace() function while inserting a single element, that function has no effects.
1. If an exception is thrown by a push_back(), push_front(), emplace_back(), or emplace_front() function, that function has no effects.
1. No erase(), clear(), pop_back() or pop_front() function throws an exception.
1. No copy constructor or assignment operator of a returned iterator throws an exception.
1. No swap() function throws an exception.
1. No swap() function invalidates any references, pointers, or iterators referring to the elements of the containers being swapped.  
  [Note 4: The end() iterator does not refer to any element, so it can be invalidated. — end note]

[\[swappable.requirements\]]: http://eel.is/c++draft/swappable.requirements

## [\[containers.sequences.general\]]

[\[containers.sequences.general\]]: http://eel.is/c++draft/sequences.general
    
Modify:

[1](http://eel.is/c++draft/sequences.general#1) The headers `<array>`, `<deque>`, `<forward_list>`, <ins>`<inplace_vector>`, </ins>`<list>`, and <vector> define class templates that meet the requirements for sequence containers.
    
## [\[container.requirements.sequence.reqmts\]]

[\[container.requirements.sequence.reqmts\]]: http://eel.is/c++draft/sequence.reqmts
    
Modify:

[sequence.reqmts.1](http://eel.is/c++draft/sequence.reqmts#1) A sequence container organizes a finite set of objects, all of the same type, into a strictly linear arrangement. The library provides <del>four</del><ins>the following</ins> basic kinds of sequence containers: `vector`,<ins> `inplace_vector`,</ins> `forward_list`, `list`, and `deque`. In addition, `array` is provided as a sequence container which provides limited sequence operations because it has a fixed number of elements. The library also provides container adaptors that make it easy to construct abstract data types, such as `stacks`, `queues`, `flat_maps`, `flat_multimaps`, `flat_sets`, or `flat_multisets`, out of the basic sequence container kinds (or out of other program-defined sequence containers).

[sequence.reqmts.2](http://eel.is/c++draft/sequence.reqmts#2) [Note [1](http://eel.is/c++draft/sequence.reqmts#note-1): The sequence containers offer the programmer different complexity trade-offs. `vector` is appropriate in most circumstances. `array` has a fixed size known during translation. <ins> `inplace_vector` has a fixed capacity known during translation.</ins> `list` or `forward_list` support frequent insertions and deletions from the middle of the sequence. `deque` supports efficient insertions and deletions taking place at the beginning or at the end of the sequence. When choosing a container, remember `vector` is best; leave a comment to explain if you choose from the rest! — end note]

[sequence.reqmts.69](http://eel.is/c++draft/sequence.reqmts#69) The following operations are provided for some types of sequence containers but not others. An implementation shall implement them so as to take amortized constant time.

```cpp
a.front()
```
* Result: `reference`; `const_reference` for constant `a`.
* Returns: `*a.begin()`
* Remarks: Required for `basic_string`, `array`, `deque`, `forward_list`,<ins> `inplace_vector`,</ins>`list`, and vector.

```cpp
a.back()
```
* Result: `reference`; `const_reference` for constant `a`.
* Effects: Equivalent to:
    ```cpp
    auto tmp = a.end();
    --tmp;
    return *tmp;
    ```
* Remarks: Required for `basic_string`, `array`, `deque`,<ins> `inplace_vector`,</ins> `list`, and vector.

```cpp
a.emplace_front(args)
```
* Result: `reference`
* Preconditions: `T` is `Cpp17EmplaceConstructible` into `X` from `args`.
* Effects: Prepends an object of type `T` constructed with `std::forward<Args>(args)...``.
* Returns: `a.front()``.
* Remarks: Required for `deque`, `forward_list`, and `list`.

```cpp
a.emplace_back(args)
```
<bdi>**EDITORIAL**: `inplace_vector` is never reallocated, so there is no need to extend the "For vector, T is also Cpp17MoveInsertable into X" to inplace_vector.
</bdi>

<bdi>**EDITORIAL**: It's okay to use `Cpp17MoveInsertable` here, even though `inplace_vector` isn’t allocator-aware. [container.alloc.reqmts.2] states: “If `X` is not allocator-aware or is a specialization of `basic_string`, the terms below [including `Cpp17MoveInsertable`] are defined as if `A` were allocator<T>”. </bdi>
    
* Result: reference
* Preconditions: `T` is `Cpp17EmplaceConstructible` into `X` from `args`. For `vector`, `T` is also `Cpp17MoveInsertable` into `X`.
* Effects: Appends an object of type `T` constructed with `std::forward<Args>(args)...`.
* Returns: `a.back()``.
* Remarks: Required for `deque`,<ins> `inplace_vector`,</ins> `list`, and vector.

```cpp
a.push_front(t)
```
* Result: `void`
* Preconditions: `T` is `Cpp17CopyInsertable` into `X`.
* Effects: Prepends a copy of `t`.
* Remarks: Required for `deque`, `forward_list`, and `list`.

```cpp
a.push_front(rv)
```
* Result: `void`
* Preconditions: `T` is `Cpp17MoveInsertable` into `X`.
* Effects: Prepends a copy of `rv`.
* Remarks: Required for `deque`, `forward_list`, and `list`.

```cpp
a.prepend_range(rg)
```
* Result: `void`
* Preconditions: `T` is `Cpp17EmplaceConstructible` into `X` from ``*ranges::begin(rg)``.
* Effects: Inserts copies of elements in `rg` before `begin()`. Each iterator in the range `rg` is dereferenced exactly once. [Note 3: The order of elements in `rg` is not reversed. — end note]
* Remarks: Required for `deque`, `forward_list`, and `list`.

```cpp
a.push_back(t)
```
* Result: Reference
* Returns: `a.back()`.
* Preconditions: `T` is `Cpp17CopyInsertable` into `X`.
* Effects: Appends a copy of `t`.
* Remarks: Required for `basic_string`, `deque`,<ins> `inplace_vector`,</ins> `list`, and vector.

```cpp
a.push_back(rv)
```
* Result: Reference
* Returns: `a.back()`
* Preconditions: `T` is `Cpp17MoveInsertable` into `X`.
* Effects: Appends a copy of `rv`.
* Remarks: Required for `basic_string`, `deque`,<ins> `inplace_vector`, </ins> `list`, and vector.

```cpp
a.append_range(rg)
```
* Result: `void`
* Preconditions: `T` is `Cpp17EmplaceConstructible` into `X` from `*ranges::begin(rg)`. For `vector`, `T` is also `Cpp17MoveInsertable` into `X`.
* Effects: Inserts copies of elements in `rg` before `end()`. Each iterator in the range `rg` is dereferenced exactly once.
* Remarks: Required for `deque`,<ins>`inplace_vector`, </ins> `list`, and vector.

```cpp
a.pop_front()
```
* Result: `void`
* Preconditions: `a.empty()` is `false`.
* Effects: Destroys the first element.
* Remarks: Required for `deque`, `forward_list`, and `list`.

```cpp
a.pop_back()
```
* Result: `void`
* Preconditions: `a.empty()` is `false`.
* Effects: Destroys the last element.
* Remarks: Required for `basic_string`, `deque`,<ins>`inplace_vector`, </ins> `list`, and vector.

```cpp
a[n]
```
* Result: `reference`; `const_reference` for constant `a`
* Returns: `*(a.begin() + n)`
* Remarks: Required for `basic_string`, `array`, `deque`,<ins> `inplace_vector`, </ins> and vector.

```cpp
a.at(n)
```
* Result: `reference`; `const_reference` for constant `a`
* Returns: `*(a.begin() + n)`
* Throws: `out_of_range` if `n >= a.size()`.
* Remarks: Required for `basic_string`, `array`, `deque`,<ins> `inplace_vector`, </ins> and `vector`.

## [containers.sequences.inplace.vector.syn] Header `<inplace_vector>` synopsis

<bdi>**Drafting note**: not freestanding yet.</bdi>
    
```cpp=
// mostly freestanding
#include <compare>              // see [compare.syn]
#include <initializer_list>     // see [initializer.list.syn]

namespace std {

// [inplace.vector], class template inplace_vector
template <class T, size_t N>
class inplace_vector; // partially freestanding

// [inplace.vector.erasure], erasure
template<class T, size_t N, class U>
constexpr typename inplace_vector<T, N>::size_type
  erase(inplace_vector<T, N>& c, const U& value);
template<class T, size_t N, class Predicate>
constexpr typename inplace_vector<T, N>::size_type
  erase_if(inplace_vector<T, N>& c, Predicate pred);

}  // namespace std
```

##  [containers.sequences.inplace.vector] Class template `inplace_vector`

### [containers.sequences.inplace.vector.overview] Overview

1. An `inplace_vector` is a contiguous container. Its capacity is fixed and part of its type, and its elements are stored within the `inplace_vector` object itself.
2. An `inplace_vector` meets all of the requirements of a container ([\[container.requirements\]]), of a reversible container ([\[container.rev.reqmts\]](http://eel.is/c++draft/container.rev.reqmts)), of a [contiguous container](http://eel.is/c++draft/container.reqmts#def:container,contiguous), and of a sequence container, including most of the optional sequence container requirements ([\[sequence.reqmts\]]). The exceptions are the `push_front`, `prepend_range`, `pop_front`, and `emplace_front` member functions, which are not provided. Descriptions are  provided here only for operations on `inplace_vector` that are not described in one of these tables or for operations where there is additional semantic information.
4. The types `iterator` and `const_iterator` meet the constexpr iterator
requirements ([\[iterator.requirements.general\]]).

[\[container.requirements\]]: http://eel.is/c++draft/container.requirements
[\[sequence.reqmts\]]: http://eel.is/c++draft/sequence.reqmts
[\[container.requirements.general\]]: http://eel.is/c++draft/container.requirements.general
[\[class.default.ctor\]]: http://eel.is/c++draft/class.default.ctor
[\[class.dtor\]]: http://eel.is/c++draft/class.dtor
[\[class.copy.ctor\]]: http://eel.is/c++draft/class.copy.ctor
[\[iterator.requirements.general\]]: http://eel.is/c++draft/iterator.requirements.general 

```cpp=
template <class T, size_t N>
class inplace_vector {
public:
// types:
using value_type             = T;
using pointer                = T*;
using const_pointer          = const T*;
using reference              = value_type&;
using const_reference        = const value_type&;
using size_type              = size_t;                 
using difference_type        = ptrdiff_t;             
using iterator               = implementation-defined; // see [container.requirements]
using const_iterator         = implementation-defined; // see [container.requirements]
using reverse_iterator       = std::reverse_iterator<iterator>;
using const_reverse_iterator = std::reverse_iterator<const_iterator>;

// [containers.sequences.inplace.vector.cons], construct/copy/destroy
constexpr inplace_vector() noexcept;
constexpr explicit inplace_vector(size_type n); // freestanding-deleted
constexpr inplace_vector(size_type n, const T& value); // freestanding-deleted
template <class InputIterator>
  constexpr inplace_vector(InputIterator first, InputIterator last); // freestanding-deleted
template <container-compatible-range<T> R>
  constexpr inplace_vector(from_range_t, R&& rg); // freestanding-deleted
constexpr inplace_vector(const inplace_vector&);
constexpr inplace_vector(inplace_vector&&) noexcept(N == 0 || is_nothrow_move_constructible_v<T>);
constexpr inplace_vector(initializer_list<T> il); // freestanding-deleted
constexpr ~inplace_vector();
constexpr inplace_vector& operator=(const inplace_vector& other);
constexpr inplace_vector& operator=(inplace_vector&& other) noexcept(N == 0 || is_nothrow_move_assignable_v<T>);
constexpr inplace_vector& operator=(initializer_list<T>); // freestanding-deleted
template <class InputIterator>
  constexpr void assign(InputIterator first, InputIterator last); // freestanding-deleted
template<container-compatible-range<T> R>
  constexpr void assign_range(R&& rg); // freestanding-deleted
constexpr void assign(size_type n, const T& u); // freestanding-deleted
constexpr void assign(initializer_list<T> il); // freestanding-deleted

// iterators
constexpr iterator               begin()         noexcept;
constexpr const_iterator         begin()   const noexcept;
constexpr iterator               end()           noexcept;
constexpr const_iterator         end()     const noexcept;
constexpr reverse_iterator       rbegin()        noexcept;
constexpr const_reverse_iterator rbegin()  const noexcept;
constexpr reverse_iterator       rend()          noexcept;
constexpr const_reverse_iterator rend()    const noexcept;

constexpr const_iterator         cbegin()  const noexcept;
constexpr const_iterator         cend()    const noexcept;
constexpr const_reverse_iterator crbegin() const noexcept;
constexpr const_reverse_iterator crend()   const noexcept;

// [containers.sequences.inplace.vector.members] size/capacity
[[nodiscard]] constexpr bool empty() const noexcept;
constexpr size_type size() const noexcept;
static constexpr size_type max_size() noexcept;
static constexpr size_type capacity() noexcept;
constexpr void resize(size_type sz); // freestanding-deleted
constexpr void resize(size_type sz, const T& c); // freestanding-deleted
static constexpr void reserve(size_type n); // freestanding-deleted
static constexpr void shrink_to_fit();

// element access
constexpr reference       operator[](size_type n);
constexpr const_reference operator[](size_type n) const;
constexpr const_reference at(size_type n) const; // freestanding-deleted
constexpr reference       at(size_type n); // freestanding-deleted
constexpr reference       front();
constexpr const_reference front() const;
constexpr reference       back();
constexpr const_reference back() const;
    
// [containers.sequences.inplace.vector.data], data access
constexpr       T* data()       noexcept;
constexpr const T* data() const noexcept;

// [containers.sequences.inplace.vector.modifiers], modifiers
template <class... Args> constexpr T& emplace_back(Args&&... args); // freestanding-deleted
constexpr T& push_back(const T& x); // freestanding-deleted
constexpr T& push_back(T&& x); // freestanding-deleted
template<container-compatible-range<T> R>
  constexpr void append_range(R&& rg); // freestanding-deleted
constexpr void pop_back();

template<class... Args>
  constexpr T* try_emplace_back(Args&&... args);
constexpr T* try_push_back(const T& x);
constexpr T* try_push_back(T&& x);
template<container-compatible-range<T> R>
  constexpr ranges::borrowed_iterator_t<R> try_append_range(R&& rg);
    
template<class... Args>
  constexpr T& unchecked_emplace_back(Args&&... args);
constexpr T& unchecked_push_back(const T& x);
constexpr T& unchecked_push_back(T&& x);
    
template <class... Args>
  constexpr iterator emplace(const_iterator position, Args&&... args); // freestanding-deleted
constexpr iterator insert(const_iterator position, const T& x); // freestanding-deleted
constexpr iterator insert(const_iterator position, T&& x); // freestanding-deleted
constexpr iterator insert(const_iterator position, size_type n, const T& x); // freestanding-deleted
template <class InputIterator>
  constexpr iterator insert(const_iterator position, InputIterator first, InputIterator last); // freestanding-deleted
template<container-compatible-range<T> R>
  constexpr iterator insert_range(const_iterator position, R&& rg); // freestanding-deleted
constexpr iterator insert(const_iterator position, initializer_list<T> il); // freestanding-deleted
constexpr iterator erase(const_iterator position);
constexpr iterator erase(const_iterator first, const_iterator last);
constexpr void swap(inplace_vector& x)
  noexcept(N == 0 || (is_nothrow_swappable_v<T> && is_nothrow_move_constructible_v<T>));
constexpr void clear() noexcept;
    
constexpr friend bool operator==(const inplace_vector& x, const inplace_vector& y);
constexpr friend synth-three-way-result<T>
  operator<=>(const inplace_vector& x, const inplace_vector& y);
constexpr friend void swap(inplace_vector& x, inplace_vector& y)
    noexcept(N == 0 || (is_nothrow_swappable_v<T> && is_nothrow_move_constructible_v<T>))
  { x.swap(y); }
};
```
    
4. Any member function of `inplace_vector<T, N>` that would cause the size to exceed `N` throws an exception of type `bad_alloc`.
    
### [containers.sequences.inplace.vector.cons] Constructors
    
    
Let `IV` denote a specialization of `inplace_vector<T, N>`.
- If `is_trivially_copy_constructible_v<T>` is true, then `IV` has a trivial copy constructor.
- If `is_trivially_move_constructible_v<T>` is true, then `IV` has a trivial move constructor.
- If `is_trivially_destructible_v<T>` is true, then `IV` has a trivial destructor.
- If `is_trivially_copy_constructible_v<T> && is_trivially_copy_assignable_v<T> && is_trivially_destructible_v<T>` is true, then `IV` has a trivial copy assignment operator.
- If `is_trivially_move_constructible_v<T> && is_trivially_move_assignable_v<T> && is_trivially_destructible_v<T>` is true, then `IV` has a trivial move assignment operator.

Furthermore, if `N==0`, then `IV` is both trivial and empty.

---

```cpp
constexpr explicit inplace_vector(size_type n);
```

- _Preconditions_: `T` is `Cpp17DefaultInsertable` into `*this`.
- _Effects_: Constructs an `inplace_vector` with `n` default-inserted elements.
- _Complexity_: Linear in `n`.
- _Throws_: `bad_alloc` if `n > capacity()` is true.

---

```cpp
constexpr inplace_vector(size_type n, const T& value);
```

- _Preconditions_: `T` is `Cpp17CopyInsertable` into `*this`.
- _Effects_: Constructs an `inplace_vector` with `n` copies of `value`.
- _Complexity_: Linear in `n`.
- _Throws_: `bad_alloc` if `n > capacity()` is true.

---

```cpp
template <class InputIterator>
constexpr inplace_vector(InputIterator first, InputIterator last);
```
    
- _Effects_: Constructs an `inplace_vector` equal to the range `[first, last)`.
- _Complexity_: Linear in `distance(first, last)`.
- _Throws_: `bad_alloc` if `distance(first, last) > capacity()` is true.
   
---

```cpp
template <container-compatible-range<T> R>
constexpr inplace_vector(from_range_t, R&& rg);
```
    
<bdi>**EDITORIAL**: could this just be `{ insert_range(end(), forward<R>(rg)); }` ? `insert_range` does not specify a _Throws_ clause, so not sure.</bdi>
<bdi>**EDITORIAL**: does `inplace_vector` need an `insert_range` that adds _Throws_</bdi>
<bdi>TODO: recommendation, move these to seq container reqs</bdi>

- _Effects_: Constructs an `inplace_vector` object with the elements of the range `rg`.
- _Complexity_: Initializes exactly _N_ elements from the results of dereferencing successive iterators of `rg`, where _N_ is `ranges::distance(rg)`.
- _Throws_: `bad_alloc` if `ranges::distance(rg) > capacity()` is true.

### [containers.sequences.inplace.vector.capacity] Size and capacity

```cpp
static constexpr size_type capacity() noexcept
static constexpr size_type max_size() noexcept
```

- _Returns_: `N`.

---

```cpp
constexpr void resize(size_type sz);
```

- _Preconditions_: `T` is _Cpp17DefaultInsertable_ into `*this`.
- _Effects_: If `sz < size()`, erases the last `size() - sz` elements from the sequence. Otherwise, appends `sz - size()` default-inserted elements to the sequence.
- _Throws_: `bad_alloc` if `sz > capacity()` is true.

<bdi>**EDITORIAL**: are we missing Remarks here?</bdi>

---

```cpp
constexpr void resize(size_type sz, const T& c);
```

- _Preconditions_: `T` is _Cpp17CopyInsertable_ into `*this`.
- _Effects_: If `sz < size()`, erases the last `size() - sz` elements from the sequence. Otherwise, appends `sz - size()` copies of `c` to the sequence.
- _Throws_: `bad_alloc` if `sz > capacity()` is true.
    
<bdi>**EDITORIAL**: are we missing Remarks here? (e.g. `vector` says: "If an exception is thrown, there are no effects")</bdi>

### [containers.sequences.inplace.vector.data] Data

```cpp
constexpr       T* data()       noexcept;
constexpr const T* data() const noexcept;
```

- _Returns_: A pointer such that `[data(), data() + size())` is a valid range. For a non-empty `inplace_vector`, `data() == addressof(front())`.
- _Complexity_: Constant time.

### [containers.sequences.inplace.vector.modifiers] Modifiers

```cpp
constexpr iterator insert(const_iterator position, const T& x); 
constexpr iterator insert(const_iterator position, T&& x);
constexpr iterator insert(const_iterator position, size_type n, const T& x);
template <class InputIterator>
  constexpr iterator insert(const_iterator position, InputIterator first, InputIterator last);
template <container-compatible-range<T> R>
  constexpr iterator insert_range(const_iterator position, R&& rg);
constexpr iterator insert(const_iterator position, initializer_list<T> il);
 
template <class... Args> constexpr iterator emplace_back(Args&&... args);
template <class... Args> constexpr iterator emplace(const_iterator position, Args&&... args);
constexpr T& push_back(const T& x);
constexpr T& push_back(T&& x);
template<container-compatible-range<T> R>
  constexpr void append_range(R&& rg);
```

- _Complexity_: linear in the number of elements inserted plus the distance to the end of the vector.
- _Throws_: `bad_alloc` if the number of elements inserted plus the vector size before exception exceeds the vector capacity.
- _Remarks_: If an exception is thrown other than by the copy constructor, move constructor, assignment operator, or move assignment operator of `T` or by any `InputIterator` operation there are no effects. If an exception is thrown while inserting a single element at the end and `T` is `Cpp17CopyInsertable` or `is_nothrow_move_constructible_v<T>` is `true`, there are no effects. Otherwise, if an exception is thrown by the move constructor of a non-`Cpp17CopyInsertable` `T`, the effects are unspecified. 

<bdi>**EDITORIAL**: `push_back` return a reference to the added element. Do we need to spell that?</bdi>
    
---

```cpp
template <class... Args>
  constexpr T* try_emplace_back(Args&&... x);
constexpr T* try_push_back(const T& x);
constexpr T* try_push_back(T&& x);
```

- _Preconditions_: `T` is _Cpp17EmplaceConstructible_ or _Cpp17MoveInsertable_ from `x`.
- _Effects_: If `size() < capacity()` is true, inserts an element constructed from its inputs at the end. Otherwise, leaves inputs unchanged.
- _Returns_: `nullptr` if `size() == capacity()` is true, and a pointer to the inserted element otherwise. 
- _Complexity_: Constant.
- _Throws_: Nothing unless an exception is thrown by the copy constructor or move constructor of `T`.
- _Remarks_: If an exception is thrown while inserting a single element at the end and `T` is `Cpp17CopyInsertable` or `is_nothrow_move_constructible_v<T>` is `true`, there are no effects. Otherwise, if an exception is thrown by the move constructor of a non-`Cpp17CopyInsertable` `T`, the effects are unspecified. 

---
                        

```cpp
template <container-compatible-range<T> R>
  constexpr ranges::borrowed_iterator_t<R> try_append_range(R&& rg);
```
                                    
- _Preconditions_: `T` is _Cpp17EmplaceConstructible_ from `*ranges::begin(x)`.
- _Effects_: Inserts copies of initial elements in `rg` before `end()`, until all elements are inserted or `size() == capacity` is `true`. Each iterator pointing to the inserted element of `rg` is dereferenced exactly once.
- _Returns_: Iterator past last inserted element of `rg`.
- _Complexity_: Linear in the number of elements inserted.  
- _Remarks_: if an exception is thrown, succesfully inserted elements are kept.
                                     
---

```cpp
template <class... Args>
  constexpr T& unchecked_emplace_back(Args&&... x);
constexpr T& unchecked_push_back(const T& x);
constexpr T& unchecked_push_back(T&& x);
```

- _Preconditions_: `size() < capacity()` is true and `T` is _Cpp17EmplaceConstructible_ or _Cpp17MoveInsertable_ from `x`.
- _Effects_: Inserts an element constructed from its inputs at the end.
- _Returns_: reference to the inserted element. 
- _Complexity_: Constant.
- _Throws_: Nothing unless an exception is thrown by the copy constructor or move constructor of `T`.
- _Remarks_: If an exception is thrown while inserting a single element at the end and `T` is `Cpp17CopyInsertable` or `is_nothrow_move_constructible_v<T>` is `true`, there are no effects. Otherwise, if an exception is thrown by the move constructor of a non-`Cpp17CopyInsertable` `T`, the effects are unspecified. 

---
                                       
   
```cpp
constexpr void reserve(size_type n);
```
    
- _Effects_: none.
- _Throws_: `bad_alloc` if `n > capacity()`.
    
---
    
```cpp
constexpr void shrink_to_fit();
```
    
- _Effects_: none.
- _Throws_: nothing.
    
---

```cpp
constexpr iterator erase(const_iterator position);
```

- _Effects_: Removes the element at `position`, destroys it, and invalidates references to elements after `position`.
- _Preconditions_: `position` in range `[begin(), end())` is true.
- _Complexity_: Linear in `size()`.
- _Remarks_: If an exception is thrown by `T`'s move constructor the effects are _unspecified_.

---

```cpp
constexpr iterator erase(const_iterator first, const_iterator last);
```

- _Effects_: Removes the elements in range `[first, last)`, destroying them, and invalidating references to elements after `last`.
- _Preconditions_: `[first, last)` in range `[begin(), end())` is true.
- _Complexity_: Linear in `size()` and `distance(first, last)`.
- _Remarks_: If an exception is thrown by `T`'s move constructor the effects are _unspecified_.
    
### [containers.sequences.inplace.vector.erasure] Erasure


---

```cpp
template<class T, size_t N, class U>
  constexpr typename inplace_vector<T, N>::size_type
    erase(inplace_vector<T, N>& c, const U& value);
```

- _Effects_: Equivalent to:
```cpp=
auto it = remove(c.begin(), c.end(), value);
auto r = distance(it, c.end());
c.erase(it, c.end());
return r;
```

---

```cpp
template<class T, size_t, class Predicate>
  constexpr typename inplace_vector<T, N>::size_type
    erase_if(inplace_vector<T, N>& c, Predicate pred);
```
    
- _Effects_: Equivalent to:
    
```cpp=
auto it = remove_if(c.begin(), c.end(), pred);
auto r = distance(it, c.end());
c.erase(it, c.end());
return r;
```
    
### [containers.sequence.inplace.vector.zero] Zero-sized inplace vector

<bdi>**EDITORIAL**: the _intent_ of any wording in this section is to guarantee that, when an `inplace_vector<T, 0>` is used as a `no_unique_address` member of a `class`, it does not increase the size of the `class`.</bdi>
    
If `IV0`, then
* `begin() == end() == ` unique value,
* `data()` return value is unspecified,
* the effect of calling `front()` or `back()` for zero-sized inplace vector is undefined, and
* member function `swap()` shall have a non-throwing exception specification.
    
## [\[version.syn\]]

[\[version.syn\]]: http://eel.is/c++draft/version.syn
    
Add:

```cpp
#define __cpp_lib_inplace_vector   202306L // also in <inplace_vector>
```

## [\[diff.cpp03.library\]] Compatibility
    
Modify:
    
* [2](https://eel.is/c++draft/diff.cpp03.library#2) Affected subclause: [\[headers\]](https://eel.is/c++draft/headers)
* Change: New headers.
* Rationale: New functionality.
* Effect on original feature: The following C++ headers are new: `<array>`, `<atomic>`, `<chrono>`, `<codecvt>`, `<condition_variable>`, `<forward_list>`, `<future>`, `<initializer_list>`,<ins>`<inplace_vector>`,</ins> `<mutex>`, `<random>`, `<ratio>`, `<regex>`, `<scoped_allocator>`, `<system_error>`, `<thread>`, `<tuple>`, `<typeindex>`, `<type_traits>`, `<unordered_map>`, and `<unordered_set>`. In addition the following C compatibility headers are new: `<cfenv>`, `<cinttypes>`, `<cstdint>`, and `<cuchar>`. Valid C++ 2003 code that `#includes` headers with these names may be invalid in this revision of C++.
    
[\[diff.cpp03.library\]]: https://eel.is/c++draft/diff.cpp03.library



# Acknowledgments

This proposal is based on Boost.Container's `boost::container::static_vector`, mainly authored by Adam Wulkiewicz, Andrew Hundt, and Ion Gaztanaga. The reference implementation is based on Howard Hinnant `std::vector` implementation in libc++ and its test-suite. The following people provided valuable feedback that influenced some aspects of this proposal: Walter Brown, Zach Laine, Rein Halbersma, Andrzej Krzemieński, Casey Carter and many others. Many thanks to Daniel Krügler for reviewing the wording.

<!-- Links -->
[stack_alloc]: https://howardhinnant.github.io/stack_alloc.html
[fixed_capacity_vector]: http://github.com/gnzlbg/fixed_capacity_vector
[boost_static_vector]: http://www.boost.org/doc/libs/1_59_0/doc/html/boost/container/static_vector.html
[travis-shield]: https://img.shields.io/travis/gnzlbg/embedded_vector.svg?style=flat-square
[travis]: https://travis-ci.org/gnzlbg/embedded_vector
[coveralls-shield]: https://img.shields.io/coveralls/gnzlbg/embedded_vector.svg?style=flat-square
[coveralls]: https://coveralls.io/github/gnzlbg/embedded_vector
[docs-shield]: https://img.shields.io/badge/docs-online-blue.svg?style=flat-square
[docs]: https://gnzlbg.github.io/embedded_vector
[folly]: https://github.com/facebook/folly/blob/master/folly/docs/small_vector.md
[eastl]: https://github.com/questor/eastl/blob/master/fixed_vector.h#L71
[eastldesign]: https://github.com/questor/eastl/blob/master/doc/EASTL%20Design.html#L284
[clump]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0274r0.pdf
[boostsmallvector]: http://www.boost.org/doc/libs/master/doc/html/boost/container/small_vector.html
[llvmsmallvector]: http://llvm.org/docs/doxygen/html/classllvm_1_1SmallVector.html
[contiguous_container]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0494r0.pdf
[constexpr_vector_1]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0597r0.html
[constexpr_vector_2]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0639r0.html
