---
layout: default
title: Spring Boot with Kotlin (2025)
date: 2025-08-10
is_index: true
description: "Best Book"
---

# Spring Boot with Kotlin (2025)
{: .no_toc }

Hello~
{: .fs-6 .fw-300 }

{% assign sorted_pages = site.books | sort: 'title' %}
{% assign doc_pages = sorted_pages | where_exp: "page", "page.url contains '/spring-boot-with-kotlin-2025/'" %}

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
