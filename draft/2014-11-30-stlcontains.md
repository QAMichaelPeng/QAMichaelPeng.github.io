---
title: stl contains compare
layout: post
---

<table>
<tr>
<th>Feature</th>
<th>array</th>
<th>vector</th>
<th>forward_list</th>
<th>list</th>
<th>queue</th>
<th>dequeue</th>
<th>priority_queue</th>
<th>stack</th>
</tr>
<tr>
<td>default constructor</td>
<td></td>
<td></td>
<td>
{% highlight c++ linenos tabsize=4 %}
forward_list() : forward_list( Allocator() ) {}
explicit forward_list( const Allocator& alloc );
{% endhighlight %}
</td>
<td></td>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
<tr>
<td>fill constructor</td>
<td>N/A</td>
<td>

{% highlight c++ linenos tabsize=4 %}
explicit vector (size_type n, 
    const allocator_type& alloc = allocator_type());
vector (size_type n, 
    const value_type& val,
    const allocator_type& alloc = allocator_type());
{% endhighlight %}
</td>
<td>

{% highlight c++ linenos tabsize=4 %}
forward_list( size_type count, 
              const T& value,
              const Allocator& alloc = Allocator());
explicit forward_list( size_type count, const Allocator& alloc = Allocator() );
{% endhighlight %}
</td>
<td></td>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
<tr>
<td>range constructor</td>
<td>N/A</td>
<td>
{% highlight c++ linenos tabsize=4 %}
template< class InputIt >
vector( InputIt first, InputIt last, 
        const Allocator& alloc = Allocator() );
{% endhighlight %}
</td>
<td>
{% highlight c++ linenos tabsize=4 %}
template< class InputIt >
forward_list( InputIt first, InputIt last, 
              const Allocator& alloc = Allocator() );
{% endhighlight %}
</td>
<td></td>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
<tr>
<td>copy constructor</td>
<td></td>
<td>
{% highlight c++ linenos tabsize=4 %}
vector( const vector& other );
vector( const vector& other, const Allocator& alloc );
{% endhighlight %}
</td>
<td>
{% highlight c++ linenos tabsize=4 %}
forward_list( const forward_list& other );
forward_list( const forward_list& other, const Allocator& alloc );
{% endhighlight %}
</td>
<td></td>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
<tr>
<td>move constructor</td>
<td></td>
<td>
{% highlight c++ linenos tabsize=4 %}
vector( const vector&& other );
vector( vector&& other, const Allocator& alloc );
{% endhighlight %}
</td>
<td>

</td>
<td></td>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
<tr>
<td>initializer constructor</td>
<td></td>
<td>
{% highlight c++ linenos tabsize=4 %}
vector( std::initializer_list<T> init, 
        const Allocator& alloc = Allocator() );
{% endhighlight %}
</td>
<td></td>
<td></td>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
</table>
