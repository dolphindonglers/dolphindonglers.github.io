---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

title: Dolphin Donglers™
layout: default
permalink: /
---

RSI 2025's favorite cult.

More blog posts coming soon! Stay tuned, and maybe subscribe to the RSS feed.


---

{% for post in site.posts %}{% if post.unlisted %}{% else %}{% if post.categories contains "shitpost" %}{% else %}
  <div id="post-short">
    <a href="{{site.url}}{{site.baseurl}}{{post.url}}">
      <h3>{{post.title}}</h3>
    </a>
    <i>{{ post.author }} posted on {{ post.date | date: "%-d %b %Y" }}</i>
    <p>
      {% if post.excerpt %}
        {{ post.excerpt }}
      {% else %}
        {{ post.content }}
      {% endif %}
    </p>
    <hr style="height:1px;color:#CCCCCC;">
  </div>
{% endif %}{% endif %}{% endfor %}
