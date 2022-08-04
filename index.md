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
   <hr>
  </article>
{% endfor %}




<br>
<hr>
`This website was written in vim`
<script>if(!sessionStorage.getItem("_swa")&&document.referrer.indexOf(location.protocol+"//"+location.host)!== 0){fetch("https://counter.dev/track?"+new URLSearchParams({referrer:document.referrer,screen:screen.width+"x"+screen.height,user:"darkwire37",utcoffset:"-4"}))};sessionStorage.setItem("_swa","1");</script>
