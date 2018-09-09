---
layout: post
---

This popped up while I was working on a presentation about the importance of CPU cache and the habit of checking your assumptions. Yes! I did mash together those two subjects. Maybe soon™ I will either do a youtube video of that presentation, or write a post about it, but now I want to talk about what I believe is an interesting performance issue with vector::insert.

**TLDR version for lazy people: Microsoft's vector::insert is not as cache-friendly as it could be in some cases. It also misses the opportunity to use one single call to memmove in the case of trivially copyable data types.**

For the not so lazy people, I invite you on a journey where we get to see a glimpse of what's in the lower levels of Microsoft's STL implementation.

I made for the presentation that I mentioned before a simple experiment that was supposed to highlight the importance of the CPU cache. The test example method (full code is shown towards the end of this post) was just doing container.insert(it, EpicStruct()) at random positions in the container. It was a classic example of list vs vector and memory access patterns.

Since half of my presentation was about checking assumptions, I decided to do exactly that - not leave my assumptions unchecked. My assumptions were that it would be hard to beat vector's timing on that test due to the way memory works.

As we will see, vector::insert is implemented in an interesting way.

It all started with this line. You can consider that EpicStruct is just a struct that has a `char m_memory[4];` as it's only member, nothing special.

```
container.insert(it, EpicStruct());
```

Source code showing the just issue and work-around is here [vector_insert_perf_issue.cpp](https://github.com/AlexEne/Presentations-2016/blob/master/Memory/vector_insert_perf_issue.cpp">vector_insert_perf_issue.cpp).
Full source code including the whole example I mentioned is here [list_vs_vector.cpp](https://github.com/AlexEne/Presentations-2016/blob/master/Memory/list_vs_vector.cpp).
Let's start digging :)

The STL code included in VisualStudio 2015 does this for the case presented above:

```cpp
iterator insert(const_iterator _Where, _Ty&& _Val)
    {    // insert by moving _Val at _Where
    return (emplace(_Where, _STD move(_Val)));
    }
```

Layer 1 out of the way, Further on, emplace:

```cpp
template<class... _Valty>
    iterator emplace(const_iterator _Where, _Valty&&... _Val)
    {    // insert by moving _Val at _Where
    size_type _Off = _VIPTR(_Where) - this->_Myfirst();
    emplace_back(_STD forward<_Valty>(_Val)...);
    _STD rotate(begin() + _Off, end() - 1, end());
    return (begin() + _Off);
    }
```

Now there is a line here that caught my attention.

```cpp
    _STD rotate(begin() + _Off, end() - 1, end());
```

When I called `insert(EpicStruct())`, std::vector puts the element at the end of the vector using emplace_back(), and then calls `rotate()`. We must go deeper:

```cpp
template<class _RanIt> inline
    _RanIt _Rotate(_RanIt _First, _RanIt _Mid, _RanIt _Last,
        random_access_iterator_tag)
    {    // rotate [_First, _Last), random-access iterators
    _STD reverse(_First, _Mid);
    _STD reverse(_Mid, _Last);
    _STD reverse(_First, _Last);
    return (_First + (_Last - _Mid));
    }
```

I really admire this implementation for rotate. It is such a nice trick with the 3 reverse calls.
But again, this is fishy, we got here from `insert()`...We need go even deeper.
What does reverse do?

```cpp
        // TEMPLATE FUNCTION reverse
template<class _BidIt> inline
    void _Reverse(_BidIt _First, _BidIt _Last, bidirectional_iterator_tag)
    {    // reverse elements in [_First, _Last), bidirectional iterators
    for (; _First != _Last && _First != --_Last; ++_First)
        _STD iter_swap(_First, _Last);
    }
```

Now we digged quite a bit in Microsoft's STL implementation and we can take a break to mention that I like the fact that finnaly I can read some of their STL code and understand it. Just look at it, it even has comments that are useful, that interval is specified as it should be, without me needing to double check with a pen on paper what this method does.

**This gives us the opportunity to notice something else: reverse(_First, _Last) swaps elements starting from _First and _Last-1 while moving the two iterators towards each other.
That can't be cache-friendly, especially for big arrays where the start and end are "far" apart, and maybe _Last is not in the cache.**

However, let's not panic about it yet. Microsoft STL's STL - Stephan T. Lavavej
[knows about this](https://twitter.com/StephanTLavavej/status/695013465342083072) performance issue, and I am sure they will fix it in a reasonable timeline.
As a side-note, the 2015 implementation is way faster than the other version that I checked (Visual Studio 2012, Update 5). But in 2012 I couldn't go this deep with my investigation since the code was filled with defines, crazy names, and things that just confused me and were a pain to debug (defines mixed with templates and other horrors).

**There is also a simple workaround for this: Just don't use the r-value reference overload for vector::insert. I know, &amp;&amp; doesn't really pop out.**  

Instead of doing this:

```
container.insert(it, EpicStruct())
```

We can replace it with:

```cpp
EpicStruct tmp = EpicStruct();
container.insert(it, tmp);
```

This will call:

```cpp
    iterator insert(const_iterator _Where, const _Ty& _Val)
        {    // insert _Val at _Where
        return (_Insert_n(_Where, (size_type)1, _Val));
        }
```

Insert_n does the right thing, moving the elements this way:

```cpp
     _Ufill(_Newvec + _Whereoff, _Count,
            _STD addressof( _Val));      // add new stuff
     ++_Ncopied;
     _Umove( this->_Myfirst(), _VIPTR(_Where),
           _Newvec);      // copy prefix
     ++_Ncopied;
     _Umove( _VIPTR(_Where), this->_Mylast(),
           _Newvec + (_Whereoff + _Count));  
```

`_Umove` ends up calling `_Uninit_move` that does the thing we would expect:

```cpp
       for (; _First != _Last; ++ _Dest, ( void)++ _First)
               _Al.construct( _Dest, ( _Valty&&)* _First);
```

## More optimizations

But there's one more thing. I know that EpicStruct is a trivially copyable data type. In other words, it doesn't have a copy constructor defined. Why not just move the elements with one call to memmove? memmove works for memory locations that overlap so it suits our needs perfectly. On my github there's a dummy vector implementation that uses memmove as a test. It's used to move the whole chunk of elements that come after the insert point, one position to the right, with one memmove call, like this:

```cpp
memmove(it + 1, it, (m_Size - off)*sizeof(T));
```

As a disclaimer MyVector is by no means an example of a vector class. I just quickly coded it in order to be able to test some assumptions.

The tests I did are using this method:

```cpp
template<class T >
double test_container(size_t count )
{
    T container;
    typename T::iterator it;

    srand(42);
    Timer tmr;

    container.push_back( EpicStruct(0));

    for (size_t i = 0; i < count; ++i)
    {
        size_t pos = rand() % container.size();

        it = container.begin();
        for ( size_t p = 0; p < pos; ++p)
        {
            //Touch each element from 0 to pos by reading it in a temp. 
            //This won't get optimized away on VS2015/VS2012
            volatile char temp = (*it).m_memory[0];
            it++;
        }

        container.insert(it, EpicStruct (i)); //the slow insert
     
        //workaround for the code above
        // EpicStruct tmp = EpicStruct (i);
        // container.insert(it, tmp);
    }

    double t = tmr.elapsed();

#if _DEBUG
    //If you want you can also print or save to file the struct.
    //Just to check that they are the same in the end.
    for (it = container.begin(); it != container.end(); ++it)
        (*it).print();
    printf("\n");
#endif

    return t;
}
```

The timings for that are the following:
**100 000 elements, 4-byte each, Visual Studio 2015, /O2**:  
Elapsed time vector: 4.235 seconds - vector.insert(it, EpicStruct()) or slow insert  
Elapsed time vector: 3.397 seconds - this is with the workaround presented above with tmp  
Elapsed time list: 13.731 seconds - lists just being lists  
Elapsed time MyVector: 1.499 seconds - memmove version

**200 000 elements, 4-byte each, Visual Studio 2015, /O2**:  
Elapsed time vector: 16.921 seconds - vector.insert(it, EpicStruct()) or slow insert  
Elapsed time vector: 12.805 seconds - this is with the workaround presented above with tmp  
Elapsed time list: 34.604 seconds - lists again :)  
Elapsed time MyVector: 5.173 seconds - memmove version  

**20 000 elements, 128-byte each, Visual Studio 2015, /O2**:  
Elapsed time vector: 4.776 seconds - vector.insert(it, EpicStruct()) or slow insert  
Elapsed time vector: 1.830 seconds - this is with the workaround presented above with tmp  
Elapsed time list: 1.072 seconds - lists start paying off since cache is small  
Elapsed time MyVector: 0.752 seconds - memmove version  

I just enabled the memmove in MyVector using a define, I am sure that you need to detect that EpicStruct is a trivially copyable using proper template magic, but on the other hand, CLANG does it, and it gives a time similar to MyVector::insert implementation that uses memmove ( even faster ). If Clang can do it I am sure you guys can do it too. Just memmove things if the type doesn't have a copy constructor.

I'm happy that now we have a more readable STL. I was having a really hard time tracking this on Visual Studio 2012 version of STL. As a side note the 2015 insert version is much faster than 2012 one.

Until Microsoft will fix the issue we can just use a temporary for a speed boost if your bottleneck really is in vector::insert, or just memmove stuff by hand if we're feeling adventurous.

As I said, the full source code that includes the memmove and other things can be found on my github [list_vs_vector.cpp](https://github.com/AlexEne/Presentations-2016/blob/master/Memory/list_vs_vector.cpp).  
The minimum (and cleaner version) for testing the performance issue is here: [vector_insert_perf_issue.cpp](https://github.com/AlexEne/Presentations-2016/blob/master/Memory/vector_insert_perf_issue.cpp).

Feel free to play around with the EpicStruct's size for example, and do some experiments on your own.  
I hope you enjoyed this, and thank you for reading until the end.