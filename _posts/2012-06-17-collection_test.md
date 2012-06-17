---
layout: post
title: "collection_test"
description: ""
category: tests
tags: [C, C++, ruby]
author:
 name: A Fake Author
 site: "mailto: localhost"
location: USA
---
{% include JB/setup %}

{% comment %}
{{ page.tags | array_to_sentence_string }} 
{% endcomment %}

Test if the subscript version works.
{% assign ta = site.tags["ruby"] %}
*{{ ta | map: 'to_liquid' |map: 'title' | array_to_sentence_string }}*

Test if the iterating version works.
{% for tag in site.tags %}
 {% if tag[0] == "ruby" %}
  *{{ ta | map: 'to_liquid' |map: 'title' | array_to_sentence_string }}*
 {% endif %}
{% endfor %} 

Placeholder
