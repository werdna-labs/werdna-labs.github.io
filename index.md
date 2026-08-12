---
title: Home
---

### Network Engineer by day; Street Fighter 6 Master by night. I work mostly on HPE
### Juniper (SRX, JUNOS EX and QFX, Mist Wireless, Mist) and HPE Aruba (AOS-CX, AOS,
### Clearpass and Central). Opinions are my own. Guides are also my own. 


## Notes

<ul class="toc">
{% assign notes = site.notes | sort: "title" %}
{% for note in notes %}
  <li>
    <a href="{{ note.url | relative_url }}">{{ note.title }}</a>
    {% if note.description %}<span class="toc-desc">{{ note.description }}</span>{% endif %}
  </li>
{% endfor %}
</ul>
