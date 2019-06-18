### Latest Guides

<div class="posts">
  {% for post in site.posts limit:10 %}
    <blockquote>
      <a href="{{ post.url }}">
        {{ post.title }}
      </a>
      <p>{{ post.date }}</p>
    </blockquote>
  {% endfor %}
</div>
