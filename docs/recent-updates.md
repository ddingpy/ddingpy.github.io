---
layout: default
title: Recent Updates
nav_order: 10
description: "View the most recently updated documentation pages"
permalink: /recent-updates/
nav_exclude: true
---

# Recent Updates
{: .no_toc }

View the most recently updated documentation pages to stay current with the latest changes.
{: .fs-6 .fw-300 }

---

## Recently Modified Pages

{% assign sorted_pages = site.pages | sort: 'date' | reverse %}
{% assign doc_pages = sorted_pages | where_exp: "page", "page.url != '/404.html'" | where_exp: "page", "page.url != '/recent-updates/'" | where_exp: "page", "page.title != nil" %}

<div class="table-wrapper">
  <table>
    <thead>
      <tr>
        <th>Page</th>
        <th>Last Updated</th>
        <th>Description</th>
      </tr>
    </thead>
    <tbody>
      {% for page in doc_pages limit:20 %}
        {% if page.url != "/" and page.url != "/feed.xml" and page.url != "/sitemap.xml" %}
        <tr>
          <td>
            <a href="{{ page.url | relative_url }}">{{ page.title }}</a>
          </td>
          <td>
            {% if page.date %}
              {{ page.date | date: "%B %d, %Y" }}
            {% else %}
              {% assign file_path = page.path %}
              {% assign file_modified = site.time %}
              {{ file_modified | date: "%B %d, %Y" }}
            {% endif %}
          </td>
          <td>
            {% if page.description %}
              {{ page.description | truncate: 100 }}
            {% else %}
              <em>No description available</em>
            {% endif %}
          </td>
        </tr>
        {% endif %}
      {% endfor %}
    </tbody>
  </table>
</div>

---

## Update History by Month

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