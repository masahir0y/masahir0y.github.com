---
layout: page
title: とあるエンジニアの備忘log
tagline: Supporting tagline
---
{% include JB/setup %}

扱っているテーマは

Linux, ARM, C言語, Makefile など。

備忘録として使っています。

[Blogger版](http://masahir0y.blogspot.jp) から引っ越してきました。
コンテンツの移動は少しずつやっていくつもり。

## Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
