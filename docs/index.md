---
layout: default
title: Contents
has_toc: false
nav_order: 10
---

# Welcome!

{% assign sorted_pages = site.pages | sort: 'date' | reverse %}
{% assign doc_pages = sorted_pages %}

## Books

There are too many books!
{: .fs-6 .fw-300 }

<ul>
  {% for page in doc_pages %}
    {% if page.url != "/" and page.url != "/feed.xml" and page.url != "/sitemap.xml" and page.dir contains '/books/' and page.is_index %}
      <li>
        <a href="{{ page.url | relative_url }}">{{ page.title }}</a>
        {% if page.description %}
          - {{ page.description | truncate: 80 }}
        {% endif %}
      </li>
    {% endif %}
  {% endfor %}
</ul>

## Articles

There are too many articles!
{: .fs-6 .fw-300 }

<ul>
  {% for page in doc_pages %}
    {% assign last_name = page.name | slice: -3,3 %}
    {% if page.url != "/" and page.url != "/feed.xml" and page.url != "/sitemap.xml" and page.dir contains 'articles' and last_name == '.md' %}
      <li>
        {% assign name_only_len = page.name.size | minus: 3 %}
        {% assign name_only = page.name | slice: 0, name_only_len %}
        <a href="{{ page.url | relative_url }}">{{ page.title | default: name_only }}</a>
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
