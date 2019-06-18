<div class="posts">
### Latest Guides
  {% for post in site.posts limit:10 %}
    <blockquote>
      <a href="{{ post.url }}">
        {{ post.title }}
      </a>
      <p>{{ post.date | date_to_string }}</p>
    </blockquote>
  {% endfor %}
</div>
