---
layout: page
title: Projects
permalink: /projects/
---

Here are my projects (continuously updated):

{% for p in site.data.projects %}
### {{ p.name }}

{{ p.desc }}

**Tech:** {% for t in p.tech %}`{{ t }}` {% endfor %}

- Repo: {{ p.repo }}
{% if p.demo and p.demo != "" %}
- Demo: {{ p.demo }}
{% endif %}

---
{% endfor %}
