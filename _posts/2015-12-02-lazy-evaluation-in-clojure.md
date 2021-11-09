---
layout: post
---
A couple of weeks ago I had to debug a really mind-boggling (for me)
problem at work and it took me the better part of three days to figure
it out. A Clojure program I had created in order to crunch some data
threw a stack overflow exception provided the input was large enough.
When I say large enough, I mean around 150 megabytes in JSON format.
So not huge, but neither negligible.

Let me describe the situation. After some processing, I had a large
vector of hash maps that looked like this:

{% highlight clojure %}
[{"ab" [1 0]} {"bc" [1 1]} {"de" [1 0]} {"ab" [1 1]}]
{% endhighlight %}

So each hash map was essentially mapping a string to a vector which
contained two numbers. The catch is that there are duplicate strings
there and the task was to merge them into one single hash map that
would contain the sum of each number in the vectors. So the example
above would become:

{% highlight clojure %}
{"ab" [2 1] "bc" [1 1] "de" [1 0]}
{% endhighlight %}

“Easy enough”, I thought and proceeded to write:

{% highlight clojure %}
(defn merge-hashes-naive [coll]
  (apply merge-with (fn [a b] (map + a b))))
{% endhighlight %}

Experienced Clojure developers should already be cringing with my
naivety.

However, the program processed all the input easily and it wasn’t until
I added the next function that things got bad. After merging all the
hash maps, I needed to drop some of them and perform calculations based
on some business rules. I decided to use a list comprehension:

{% highlight clojure %}
(def simple-list-comp [results]
  (for [[k [a b]] results]
    :when ;; do stuff
{% endhighlight %}

For small input, this worked fine. It wasn’t until I tried it using
the full input that the stack overflow exception was thrown. And the
stack trace was huge:

{% highlight console %}
java.lang.StackOverflowError
     at clojure.core$seq.invoke(core.clj:133)
     at clojure.core$concat$fn__3955.invoke(core.clj:685)
     at clojure.lang.LazySeq.sval(LazySeq.java:40)
     at clojure.lang.LazySeq.seq(LazySeq.java:49)
     at clojure.lang.RT.seq(RT.java:484)
     at clojure.core$seq.invoke(core.clj:133)
     ......
{% endhighlight %}

The first thing I thought was “I saw this somewhere recently” and
quickly remembered Stuart Sierra’s [post on the concat function](http://stuartsierra.com/2015/04/26/clojure-donts-concat)
(I think his “[Clojure do’s and dont’s](http://stuartsierra.com/tag/dos-and-donts)”
series is mandatory reading for Clojure programmers). He writes:

> This is a nasty bug in production code, because it could occur far
> away from its source, and the accumulated stack frames of seq prevent
> us from seeing where the error originated.

After trying a lot of things to solve it, all to no avail — the program
had much more going on than these two functions — I decided to try to
read the bottom of the (usually truncated) stack trace. Fortunately,
there is a JVM option for that:

{% highlight console %}
-XX:MaxJavaStackTraceDepth=-1
{% endhighlight %}

Long story short, the stack trace pointed to the second line of the
`simple-list-comp` function. I thought a bit about it, decided it was
random and ignored it completely.

A couple of days passed.

By then it was Friday, and this had taken me almost half a week to
solve. I could just use a larger stack, but knew that I would have
to deal with it eventually. So I thought I would take another look at
the `simple-list-function`. I removed all logic from it and just let it
pass once through the input. Stack overflow. Then, I changed the
destructuring:

{% highlight clojure %}
(for [[a b] results] a)
{% endhighlight %}

*It didn’t blow up.* Somehow, the destructuring of the vector was where
the problem occurred (note to self: I should trust the stack traces
more). Suddenly, it all clicked: *The destructuring triggered the
evaluation, it was all lazy seqs until that point!* But of course!
The map function wraps its result in a lazy seq, meaning that nothing
gets evaluated until the result is needed.

What this means is that whenever the call to `merge-with` encountered
a key it had seen before, instead of calculating the result, it wrapped
the calculation inside a lazy seq and returned. For example, merging:

{% highlight clojure %}
[{"ab" [1 0]} {"ab" [2 1]} {"ab" [0 1]}]
{% endhighlight %}

instead of returning just:

{% highlight clojure %}
{"ab" [3 2]}
{% endhighlight %}

would essentially return:

{% highlight clojure %}
{"ab" (map + [0 1] (map + [2 1] [1 0]))}
{% endhighlight %}

where the nested maps would only get evaluated when needed, which in my
case was when destructuring the list comprehension arguments. Each
nested call would consume a bit of the stack and when the nesting
became large enough, the stack would overflow. In my case it took about
1,500 calls (so a key would exist 1,500 times in the original
collection).

The easiest mitigation was to replace `map` with its eager counterpart,
`mapv`:

{% highlight clojure %}
(defn merge-hashes-naive [coll]
  (apply merge-with (fn [a b] (mapv + a b))))
{% endhighlight %}

But the real gain was finally understanding that lazy evaluation
happens all the time in Clojure. For example `map`, `filter`, `range`
and `take` are all lazy.

I hope this will be helpful to another Clojure newbie in the future.
For more about lazy evaluation in Clojure I suggest reading [Laziness in Clojure](http://clojure-doc.org/articles/language/laziness.html)
and chapter 3 from the excellent [Joy of Clojure](https://www.manning.com/books/the-joy-of-clojure-second-edition)
book.

