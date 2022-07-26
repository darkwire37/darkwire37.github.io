<h2>{{ site.data.navbar.navbar_title }}</h2>
<ul>
   {% for item in site.data.navbar.docs %}
      <li><a href="{{ item.url }}">{{ item.title }}</a></li>
   {% endfor %}
</ul>

# Welcome to my blog

> It's been a long time... - GLaDOS, Aperture Laboratories
I've been doing cyber for a few years now, but never blogged any of it.  That was a mistake.  So, I figured there's no time like the present to start!
There will be a really large variety of posts here, so take a seat, grab a drink, and enjoy the chaos.  I hope you learn something in your time here.

## All Posts
{% for post in site.posts %}
  <article>
    <h3>
      <a href="{{ post.url }}">
        {{ post.title }}
      </a>
    </h3>
    <time datetime="{{ post.date | date: "%Y-%m-%d" }}">{{ post.date | date_to_long_string }}</time>
   <br>
  </article>
{% endfor %}




<br>
<hr>
`This website was written in vim`
