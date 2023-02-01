---
layout: archive
permalink: categories/Elastic
title: "Elastic"
author_profile: true
sidebar_main: true
---

{% assign posts = site.categories.Elastic %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}