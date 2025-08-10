---
layout: default
title: Books
nav_order: 10
description: "View the most recently updated documentation pages"
permalink: /books/
---

# Recent Updates
{: .no_toc }

View the most recently updated documentation pages to stay current with the latest changes.
{: .fs-6 .fw-300 }

## Update History by Month

{% assign sorted_pages = site.books | sort: 'date' | reverse %}
{% assign doc_pages = sorted_pages | where_exp: "page", "page.is_index == true" | where_exp: "page", "page.title != nil" %}

{% assign pages_by_month = "" | split: "" %}
{% for page in doc_pages %}
{% if page.url != "/" and page.url != "/feed.xml" and page.url != "/sitemap.xml" %}
{% assign month_year = page.date | date: "%B %Y" | default: site.time | date: "%B %Y" %}
{% assign pages_by_month = pages_by_month | push: month_year %}
{% endif %}
{% endfor %}

{% assign unique_months = pages_by_month | uniq | sort | reverse %}

{% for month in unique_months limit:6 %}
### {{ month }}
{: .fw-500 }

<ul>
  {% for page in doc_pages %}
    {% if page.url != "/" and page.url != "/feed.xml" and page.url != "/sitemap.xml" %}
      {% assign page_month = page.date | date: "%B %Y" | default: site.time | date: "%B %Y" %}
      {% if page_month == month %}
        <li>
          <a href="{{ page.url | relative_url }}">{{ page.title }}</a>
          {% if page.description %}
            - {{ page.description | truncate: 80 }}
          {% endif %}
        </li>
      {% endif %}
    {% endif %}
  {% endfor %}
</ul>
{% endfor %}

---

<div class="fs-2 text-grey-dk-000">
  <em>Note: Update dates are based on when pages were last modified. For more accurate tracking, ensure your documentation pages include a `date` field in their front matter.</em>
</div>
