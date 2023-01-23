---
layout: archive
permalink: aws
title: "AWS"

author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories.aws %}
{% for post in posts %}
  {% include custom-archive-single.html type=entries_layout %}
{% endfor %}