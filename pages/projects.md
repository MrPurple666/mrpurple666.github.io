---
title: "Projetos"
permalink: /projects/
layout: single
header:
  image: /assets/images/pic02.jpg
---

Sou contribuidor e amante do open source e da diversidade.

Os projetos open source foram parte fundamental da minha formação. Já contribuí com custom ROMs, modificações de kernel no Android, Schrodinger-Hat e outros projetos. Também dedico parte do meu tempo como voluntário na escola onde estudo, ajudando alunos com dificuldades em programação.

Acredito em compartilhar conhecimento e ajudar outras pessoas a crescerem na área. Essa experiência tem sido gratificante e fortalece meu crescimento pessoal e profissional.

## Meus Projetos

<ul>
  {% for page in site.pages %}
    {% if page.url contains '/projects/' and page.url != '/projects/' %}
      <li><a href="{{ page.url }}">{{ page.title }}</a></li>
    {% endif %}
  {% endfor %}
</ul>

## Onde me encontrar

- [GitHub](https://github.com/MrPurple666)
- [Spotify](https://open.spotify.com/artist/6IIzCWouZ8TKK5xsC9Zyuu?si=TozHtiHFQmyOWb5RE2zEag)
