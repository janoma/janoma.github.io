---
layout: post
title: "Using disjoint sets with a vector"
tags: [boost,c++]
category: programming
---

## Motivation
After much struggling with making `boost::disjoint_sets` work in a slightly
non-trivial way, I present here a solution given the following restrictions on
the data and conditions on the final setting:

* The kind of object we are trying to separate into disjoint sets is not an
  `int`, but a `class` with some internal logic for how the sets should be
  formed.
* Moreover, we have edit access to this class' code, so we can add some
  attributes and methods if we need to.
* `std::map` is considered too expensive, so [this
  alternative](http://stackoverflow.com/a/4136546/671555)
  is not an acceptable solution. In particular, it requires an empty constructor
  for the `operator[]`, which is a no go in my context.
* The total number of objects to be separated into different sets is known in
  advance, and the objects are available *before* starting to make the sets.

## An overview of the solution
Our solution consists in using a single `std::vector` to store all the elements,
whose class is modified so that the structures needed by `disjoint_sets` are all
part of the objects themselves. We add three integers to the class, which seems
like a reasonable price to pay compared to the overhead involved in a `map`, for
example.

### The modified `Element` class
Firstly, let's say we have a class `Element`, and it's this kind of object that
we want to separate into disjoint sets. We modify the class so it has 3 new
attributes: an *ID* (which will uniquely identify the instance), a *rank* (which
helps with the union by rank), and a *parent* (which is another element of the
same set). In this case, the only other attribute is an integer, but the class
could very well be much more complex.

{% highlight c++ %}
class Element
{
public:
    explicit
    Element(int n) : mSomeInt(n) { }

    int someInt() const { return mSomeInt; }

    // Specific for disjoint_sets
    size_t dsID;
    size_t dsRank;
    size_t dsParent;

private:
    int mSomeInt;
};
{% endhighlight %}

Note how I made these additional attributes `public` for convenience. However,
you can always make them `private` and provide read and write accessors for all
of them (getters and setters, if you will).

### Comparison operators
We will need two common operators for this class: `==` and `!=`. If these are
already provided by the class, there's nothing else to do. For the purposes of
this example, I added them outside.

{% highlight c++ %}
inline bool
operator==(Element const& lhs, Element const& rhs)
{
    return lhs.someInt() == rhs.someInt();
}

inline bool
operator!=(Element const& lhs, Element const& rhs)
{
    return ! operator==(lhs, rhs);
}
{% endhighlight %}

In no case the attributes added for disjoint sets should be used in any of these
comparisons.

One more compare function is added here, used to sort the `vector` by *parent*,
so each element of the partition is contiguous.

{% highlight c++ %}
inline bool
compareByParent(Element const& lhs, Element const& rhs)
{
    return lhs.dsParent < rhs.dsParent;
}
{% endhighlight %}

### Rank and Parent
Two classes are required by `boost::disjoint_sets`: `Rank` and `Parent`. Note
that we simply define them to contain a reference to the vector that actually
contains our elements. Although they could be just one class definition, I'm
separating them for the sake of readability.

{% highlight c++ %}
class Parent
{
public:
    Parent(std::vector<Element>& e) : mElements(e) { }
    std::vector<Element>& mElements;
};

class Rank
{
public:
    Rank(std::vector<Element>& e) : mElements(e) { }
    std::vector<Element>& mElements;
};
{% endhighlight %}

As before, you could have `mElements` as a `private` member if you wish.

The `Rank` class requires this `typedef` to find some value type, and this
requires to `#include <boost/pending/property.hpp>`.

{% highlight c++ %}
template <>
struct boost::property_traits<Rank*>
{
    typedef size_t value_type;
};
{% endhighlight %}

### `get` and `put`
Both `Rank` and `Parent` require an interface to read and write the
corresponding value for a given key, which is an element of the vector.

{% highlight c++ %}
inline Element const&
get(Parent* pa, Element const& k)
{
    return pa->mElements.at(k.dsParent);
}

inline void
put(Parent* pa, Element k, Element const& val)
{
    pa->mElements.at(k.dsID).dsParent = val.dsID;
}

inline size_t const&
get(Rank*, Element const& k)
{
    return k.dsRank;
}

inline void
put(Rank* pa, Element k, size_t const& val)
{
    pa->mElements.at(k.dsID).dsRank = val;
}
{% endhighlight %}

In both versions of `put`, the second parameter is a copy of an `Element`, which
might not be optimal. So far, it has worked for me after replacing `Element k`
by `Element const& k`, but I don't know whether that could cause trouble in some
weird scenario.

## Putting it all together
It's time to test this setting. In our example program, we will reserve space
for a fixed number of `Element`s, fill it with some randomly created objects,
and then create the sets and make some unions at random as well.

{% highlight c++ %}
std::vector<Element> elements;
elements.reserve(30);
for (size_t i = 0; i < elements.capacity(); ++i)
    elements.push_back(Element(rand() % 90));

for (size_t i = 0; i < elements.size(); ++i)
    elements[i].dsID = i;

Rank rank(elements);
Parent parent(elements);

boost::disjoint_sets<Rank*, Parent*> sets(&rank, &parent);

for (size_t i = 0; i < elements.size(); ++i)
    sets.make_set(elements.at(i));

for (size_t i = 0; i < elements.size(); ++i)
{
    // Union with an element randomly chosen from the rest
    size_t j = rand() % elements.size();
    sets.union_set(elements[i], elements[j]);
}

std::cout << "Found " << sets.count_sets(elements.begin(), elements.end()) << " sets." << std::endl;
{% endhighlight %}

### Obtaining the sets
So far, we've been relying on the identifier `dsID` to be exactly the position
of the element in the vector. Once we have finished joining sets, however, it is
time to sort the elements so each resulting set is a contiguous chunk of memory.

For this, we first flatten the tree of parent identifiers, so that the parent of
each element is exactly the representative of the set. This is called path
compression, and is again provided directly by boost.

{% highlight c++ %}
sets.compress_sets(elements.begin(), elements.end());
{% endhighlight %}

Now we can sort by `dsParent`, using the function we defined before.

{% highlight c++ %}
std::sort(elements.begin(), elements.end(), compareByParent);
{% endhighlight %}

### Looping through each set
After sorting by parent, which in turn was done *after* path compression, we are
ready to loop through each set of the partition, and through each element of
each set.

#### Using indices
If we wish, we could use the inner loop to update `mID` so it reflects the new
position of the element in the vector.

{% highlight c++ %}
size_t first = 0;
size_t total = elements.size();
while (first < total)
{
    size_t parent = elements[first].dsParent;
    size_t last = first;
    while (last < total && elements.[last].dsParent == parent)
        ++last;
    // The range [first,last) is a set of the partition
    for (size_t i = first; i < last; ++i)
    { /* elements[i] belongs to the current set */ }
    first = last;
}
{% endhighlight %}

#### Using iterators
This method, in particular, requires `dsParent` to be sorted in increasing
order, so `compareByParent` will be the appropriate comparison for the
`upper_bound` algorithm.

{% highlight c++ %}
typedef std::vector<Element>::iterator Iterator;
Iterator first = elements.begin();
Iterator end = elements.end();
while (first != elements.end())
{
    Iterator last = std::upper_bound(
        first, end, *first, compareByParent);
    // The range [first,last) is a set of the partition
    for (Iterator it = first; it != last; ++it)
    { /* *it belongs to the current set */ }
    first = last;
}
{% endhighlight %}

## Conclusion
This is one specific example of usage. I hope it helps somebody somewhere, as I
know by experience how painful it can be to try and make sense of all of this
with the unfortunate documentation that boost provides.

To see the full code, you can visit the
[repository](https://github.com/janoma/study/tree/master/disjoint_sets) where I
uploaded this example.
