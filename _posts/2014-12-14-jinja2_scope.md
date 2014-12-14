---
title: variable scope issues in Jinja2
layout: post
---
I tried to calculate the sum of a sequence in a loop in jinja2 template but got some trouble. Below is my initial code:

{% highlight python linenos tabsize=4 %}
{% raw %}
from jinja2 import Template
text1 = """
{%- set sum = 100 -%}
<ul>
{%- for i in range(2) -%}
{%- set sum = sum + 1 %}
<li>{{sum}}</li>
{%- endfor %}
</ul>
{{sum}}
"""
template = Template(text1)
print template.render()
{% endraw %}
{% endhighlight %}

Here's the output:

{% highlight html linenos tabsize=4 %}
<ul>
<li>101</li>
<li>102</li>
</ul>
100
{% endhighlight %}

At the end of the loop, sum is 100, not the expected 103. I was confused at first. By doing some search, I found this may be the issue caused by variable scope. Then I tried to work around.

{% highlight python linenos tabsize=4 %}
{% raw %}
from jinja2 import Template
text2 = """
{%- set l = [0] -%}
<ul>
{%- for i in range(2) -%}
{%- if l[0] = l[0] + 1 %}
{%- endif -%}
<li>{{l[0]}}</li>
{%- endfor %}
</ul>
{{l[0]}}
"""
template = Template(text2)
print template.render()
{% endraw %}
{% endhighlight %}

Unfortunately assignment is a statement instead of an expresion in Python, unlike the assignment expression in C++. So we can't use it in if and got the following error:
{% highlight text linenos tabsize=4 %}
{% raw %}
Traceback (most recent call last):
  File "test.py", line 26, in <module>
    template = Template(text2)
  File "C:\Python27\lib\site-packages\jinja2\environment.py", line 906, in __new__
    return env.from_string(source, template_class=cls)
  File "C:\Python27\lib\site-packages\jinja2\environment.py", line 841, in from_string
    return cls.from_code(self, self.compile(source), globals, None)
  File "C:\Python27\lib\site-packages\jinja2\environment.py", line 554, in compile
    self.handle_exception(exc_info, source_hint=source)
  File "C:\Python27\lib\site-packages\jinja2\environment.py", line 742, in handle_exception
    reraise(exc_type, exc_value, tb)
  File "<unknown>", line 5, in template
jinja2.exceptions.TemplateSyntaxError: expected token 'end of statement block', got '='
{% endraw %}
{% endhighlight %}


We need to find a function to set list's element instead of the assignment expression. Fortunately we've lots of methods for that. 

{% highlight python linenos tabsize=4 %}
{% raw %}
from jinja2 import Template
text2 = """
<ul>
{%-set l = [100] %}
{%- for i in range(2) %}
{%- if l.insert(0, l[0] + i) -%}
{% endif %}
<li>{{l[0]}}</li>
{%- endfor %}
</ul>
{{l[0]}}
"""
template = Template(text2)
print template.render()
{% endraw %}
{% endhighlight %}

Now we got what we expected.
{% highlight html linenos tabsize=4 %}
{% raw %}
<ul>
<li>100</li>
<li>101</li>
</ul>
101
{% endraw %}
{% endhighlight %}

We can use dummy variable and set to replace if statement and save one line.


{% highlight python linenos tabsize=4 %}
{% raw %}
from jinja2 import Template
text2 = """
<ul>
{%-set l = [100] %}
{%- for i in range(2) %}
{%- set _ = l.insert(0, l[0] + i) %}
<li>{{l[0]}}</li>
{%- endfor %}
</ul>
{{l[0]}}
"""
template = Template(text2)
print template.render()
{% endraw %}
{% endhighlight %}


###Reference:###

[Can a Jinja variable's scope extend beyond in an inner block?](http://stackoverflow.com/questions/4870346/can-a-jinja-variables-scope-extend-beyond-in-an-inner-block)
