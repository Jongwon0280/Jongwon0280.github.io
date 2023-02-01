---
layout: archive
permalink: categories/aws
title: "AWS"
author_profile: true
sidebar_main: true
---

{% assign posts = site.categories.aws %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}