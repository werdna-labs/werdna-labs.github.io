---
title: Home
---

## Network Engineer by day; Street Fighter 6 Master by night.

I work mostly on HPE Juniper (SRX, JUNOS EX and QFX, Mist Wireless, Mist) and
HPE Aruba (AOS-CX, AOS, Clearpass and Central). Opinions are my own. Guides are
also my own. In fact, this whole thing is my own I suppose.

## Labs

<ul class="toc">
  <li>
    <a href="{{ '/labs/vxlan-packet-ride/' | relative_url }}">VXLAN Packet Ride</a>
    <span class="toc-desc">Ride a packet through an EVPN-VXLAN fabric in your browser or a VR headset.</span>
  </li>
</ul>

## Notes

{% comment %}
  Groups notes by the `section:` front matter value, in the order listed under
  `sections:` in _config.yml. Empty sections are skipped. A note with no
  section, or with a section name not in that list, lands under "Other" so
  nothing silently disappears. The dashes inside the tags strip blank lines
  and indentation so kramdown sees compact HTML, never an indented code block.
{% endcomment -%}
{%- assign notes = site.notes | sort: "title" -%}
{%- for section in site.sections -%}
  {%- assign in_section = notes | where: "section", section -%}
  {%- if in_section.size > 0 %}
<h3 class="toc-section">{{ section }}</h3>
<ul class="toc">
    {%- for note in in_section %}
<li><a href="{{ note.url | relative_url }}">{{ note.title }}</a>{% if note.description %}<span class="toc-desc">{{ note.description }}</span>{% endif %}</li>
    {%- endfor %}
</ul>
  {%- endif -%}
{%- endfor -%}
{%- assign other_count = 0 -%}
{%- for note in notes -%}
  {%- unless site.sections contains note.section -%}{%- assign other_count = other_count | plus: 1 -%}{%- endunless -%}
{%- endfor -%}
{%- if other_count > 0 %}
<h3 class="toc-section">Other</h3>
<ul class="toc">
  {%- for note in notes -%}
    {%- unless site.sections contains note.section %}
<li><a href="{{ note.url | relative_url }}">{{ note.title }}</a>{% if note.description %}<span class="toc-desc">{{ note.description }}</span>{% endif %}</li>
    {%- endunless -%}
  {%- endfor %}
</ul>
{%- endif %}
