---
title: "Projetos"
permalink: /projects/
layout: single
header:
  image: /assets/images/pic02.jpg
---

​Código Aberto e Conhecimento Livre

​O open source não é apenas um modelo de desenvolvimento; é a base da minha formação e a minha filosofia de trabalho. Construí minha trajetória desmontando e modificando sistemas, com contribuições que vão desde custom ROMs, modificações de kernel no Android e projetos como o Schrodinger-Hat.

​Atualmente, direciono grande parte da minha energia para a comunidade de emulação, colaborando com o emulador Eden e mergulhando na engenharia e otimização de drivers de vídeo, como o Turnip (Mesa3D).

​Acredito que a tecnologia só atinge seu verdadeiro potencial quando o conhecimento é acessível e construído em um ambiente diverso. Para mim, documentar descobertas, compartilhar código e colaborar com a comunidade é a melhor forma de garantir que mais pessoas tenham as ferramentas necessárias para explorar e reescrever as limitações do software.

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
