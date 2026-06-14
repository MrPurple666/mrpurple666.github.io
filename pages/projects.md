---
title: "Projects"
permalink: /projects/
layout: single
lang: en
locale: en
translation_key: projects
alternate_url: /pt/projects/
header:
  image: /assets/images/pic02.jpg
---

Open Source Code and Shared Knowledge

Open source is not just a development model for me; it is the basis of how I learned and how I work. My trajectory was built by taking systems apart and reshaping them, with contributions ranging from custom ROMs and Android kernel modifications to projects like Schrodinger-Hat.

Today, much of my energy goes into the emulation community, collaborating with the Eden emulator and diving into graphics driver engineering and optimization, especially around Turnip in Mesa3D.

I believe technology reaches its full potential only when knowledge is accessible and built in a diverse environment. For me, documenting discoveries, sharing code, and collaborating with the community is the best way to give more people the tools they need to explore and rewrite software limitations.

## My Projects

{% assign project_pages = site.pages | where: "project", true | where: "lang", page.lang | sort: "title" %}
<ul>
  {% for project in project_pages %}
    <li><a href="{{ project.url }}">{{ project.title }}</a></li>
  {% endfor %}
</ul>

## Where to find me

- [GitHub](https://github.com/MrPurple666)
- [Spotify](https://open.spotify.com/artist/6IIzCWouZ8TKK5xsC9Zyuu?si=TozHtiHFQmyOWb5RE2zEag)
