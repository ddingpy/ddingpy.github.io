---
layout: default
title: Books
nav_order: 10
permalink: /books/
---

# Books
{: .no_toc }

There are too many books!
{: .fs-6 .fw-300 }

{% assign sorted_pages = site.books | sort: 'date' | reverse %}
{% assign doc_pages = sorted_pages | where_exp: "page", "page.is_index == true" | where_exp: "page", "page.title != nil" %}

<ul>
  {% for page in doc_pages %}
    {% if page.url != "/" and page.url != "/feed.xml" and page.url != "/sitemap.xml" %}
      <li>
        <a href="{{ page.url | relative_url }}">{{ page.title }}</a>
        {% if page.description %}
          - {{ page.description | truncate: 80 }}
        {% endif %}
      </li>
    {% endif %}
  {% endfor %}
</ul>

---

<div class="fs-2 text-grey-dk-000">
  <em>Note: Update dates are based on when pages were last modified. For more accurate tracking, ensure your documentation pages include a `date` field in their front matter.</em>
</div>
