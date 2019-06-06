### Latest Guides

<div class="posts">
  {% for post in site.posts limit:10 %}
    <div class='post-ref'>
      <a href="{{ post.url }}">
        {{ post.title }}
      </a>
    </div>
  {% endfor %}
</div>
