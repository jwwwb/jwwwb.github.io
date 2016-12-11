---
layout: post
title:  "Pointers in Python."
date:   2016-12-11 19:07:44 +0100
categories: python
---
So it's been a while since I've posted something here, and the reason for that is mostly because I've
been quite busy with projects that prevented me from working on my "fun" projects, like the chess bot.
Though I realized that that's really not an excuse at all, I've made so many useful discoveries and
learnt so many new things, I'd really benefit from writing them down and structuring my thoughts a bit.

As a self-taught programmer with a non-CS background, there are a lot of things I figure out on the fly,
usually only when my lack of knowledge comes around to bite me in the butt. A good example of one such
case is the distiction between call by value and call by reference. It's actually something I've known
about, but without a lot of practical experience it was hard for me to really remember how it affects me
in practice. And these down and dirty details of programming really aren't something you worry about at
all when working with Python. For the most part you just write code how you think it, and the interpreter
makes it work somehow. And especially with duck typing, you really don't think about how a variable is 
stored or where it's located, you just care about what's inside it. And unlike C++, the last thing you
think about is pointers.

During my investigations of [Google's Pregel][pregel] distributed graph mining system, I set out to build a small
scale single machine replica of it, to get a better idea of what is going on behind the scenes, and
how global results emerge from small scale multi-agent type behavior. You write what you know, so
naturally I did it in Python, with a bit of Qt5 for the visualization. Since the real version uses
message passing for communication between workers, and my replica wasn't going to be using any real 
networking, I just used empty lists as buffers to represent the network buffer. Little did I know, this was
to be the beginning of an hour-long debugging session. 

To cut to the chase, the problem was that all of my buffer lists were the same list. So every entry I
tried to add to one, I actually added to all of them. Why this initially worked anyway, but then started causing errors, 
I don't know - those are the worst kind of errors: the ones that only arise sometimes and usually not at the point 
in time where you actually introduced the mistake into the code. By sheer luck I had recently been reading 
this excellent [blog post by Rob Heaton about references in Python][heaton-blog] which even made me think
of the fact that this could be an issue at all.

One of the things I love about Python is the overloading of operators and functions that lets you 
write code in a way that makes intuitive sense to a human. Do you want to know if two strings are
the same? Don't mess around with strcmp(), just type:

{% highlight python %}
>>> 'word' == 'word'
True
{% endhighlight %}

Want to concatenate strings together? Easy:

{% highlight python %}
>>> 'word' + 'play'
'wordplay'
{% endhighlight %}

How about concatenate a bunch of copies of a string? Just do:

{% highlight python %}
>>> 'bla' * 5
'blablablablabla'
{% endhighlight %}

The human brain has a really good (although perhaps not rigidly defined) idea of what "times" and "equal" 
mean, namely thing1 == thing2 if they are the same,  and thing1 times 3 equals thing1 + thing1 + thing1. The beauty of
Python is that everything is an object, and every object can (and probably should) define comparison
or arithmetic methods. The same goes for collections: the ability to "for item in collection" over just
about any non-atomic data-type is wonderful and makes for really readable code. I've totally forgotten how
to even write for( int i = 0; i < 10; i++ ) style loops anymore. 

One of the things I do often in Python when I need several distinct, but identically behaved, items (though 
I'm sure more experienced Python programmers than me could explain to me in what way this is non-pythonic) is create 
a list of items using multiplication:
 
{% highlight python %}
>>> counters = [0]*10
>>> counters
[0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
{% endhighlight %}

Now whenever some event happens at location i, I can simply increment counters[i] to count that occurrence. 

{% highlight python %}
>>> counters[3] += 1
>>> counters[5] += 8
>>> counters
[0, 0, 0, 1, 0, 8, 0, 0, 0, 0]
{% endhighlight %}

Wonderful. No reason why these have to be integers, I can just as easily use strings:

{% highlight python %}
>>> morse_decoders = ['']*3
>>> morse_decoders
['', '', '']
>>> morse_decoders[1] += 'SOS'
>>> morse_decoders
['', 'SOS', '']
{% endhighlight %}

Encouraged by this success, you might also try to do the same thing with lists:

{% highlight python %}
>>> incoming_messages = [[]]*3
>>> incoming_messages
[[], [], []]
>>> incoming_messages[1].append("Hi James.")
>>> incoming_messages[1].append("It's mom.")
>>> incoming_messages
[['Hi James.', "It's mom."], ['Hi James.', "It's mom."], ['Hi James.', "It's mom."]]
{% endhighlight %}

Hmmmmmm.... turns out when you initalize your lists in this way, it just replicates the pointer to
the list three times. So all three lists are the same object, changing one changes them all. The quick
fix for this is to use list comprehension instead of multiplication:

{% highlight python %}
>>> incoming_messages = [[] for _ in range(3)]
>>> incoming_messages
[[], [], []]
>>> incoming_messages[1].append("Hi James.")
>>> incoming_messages[1].append("It's mom.")
>>> incoming_messages
[[], ['Hi James.', "It's mom."], []]
{% endhighlight %}

So what happened here? Well, it's not that lists in Python are pointers and ints are not. In fact, really
everything is a pointer, from the ints to the lists. Using the id() function to reveal the memory address
of variables can be quite enlightening:

{% highlight python %}
>>> list_of_ints = range(5)
>>> list_of_ints
[0, 1, 2, 3, 4]
>>> [id(value) for value in list_of_ints]
[140373411256208, 140373411256184, 140373411256160, 140373411256136, 140373411256112]
{% endhighlight %}

Five different values, five different locations in memory. So far so good. How about this one:

{% highlight python %}
>>> list_of_ints = [5, 5, 5, 5, 5]
>>> list_of_ints
[5, 5, 5, 5, 5]
>>> [id(value) for value in list_of_ints]
[140373411256088, 140373411256088, 140373411256088, 140373411256088, 140373411256088]
{% endhighlight %}

Wait, what? All of these fives, which were initialized separately from each other, are all at the same location 
in memory? So what happens if we modify one of them?

{% highlight python %}
>>> list_of_ints[0] += 3
>>> list_of_ints
[8, 5, 5, 5, 5]
>>> [id(value) for value in list_of_ints]
[140373411256016, 140373411256088, 140373411256088, 140373411256088, 140373411256088]
{% endhighlight %}

Not only did the value of the first entry change, it's location in memory changed also! Python is smart enough
to know that if you have a lot of pointers pointing at the same integer, it's probably unlikely that you want
to change all of them when you change the value from one variable. Unfortunately, for appending to lists, it
doesn't do this (well, actually there's a very good reason for this - copy operations are expensive, and for a
potentially very long and recursive list, this is not something you want to undertake lightly)

{% highlight python %}
>>> list_of_lists = [[]] * 5
>>> [id(sublist) for sublist in list_of_lists]
[4411428592, 4411428592, 4411428592, 4411428592, 4411428592]
>>> list_of_lists[0].append(500)
>>> [id(sublist) for sublist in list_of_lists]
[4411428592, 4411428592, 4411428592, 4411428592, 4411428592]
{% endhighlight %}

So, the main thing to be aware of, is that any assignments in python simply copy the pointer of the value being
assigned. If you then want to be able to manipulate the two variables separately, make sure that they are of a 
datatype that allows this (like integers, strings, basically anything immutable). For everything else, copy() and
deepcopy() are your friends. Or you may want to take advantage of the fact that objects are passed by reference,
to allow shared reading of the same item.



[pregel]: https://blog.acolyer.org/2015/05/26/pregel-a-system-for-large-scale-graph-processing/
[heaton-blog]: http://robertheaton.com/2014/02/09/pythons-pass-by-object-reference-as-explained-by-philip-k-dick/

