---
title: "Construindo um Emulador de Game Boy do Zero"
permalink: /pt/projects/purplegb/
layout: single
lang: pt-BR
locale: pt-BR
project: true
translation_key: project-purplegb
alternate_url: /projects/purplegb/
header:
  image: /assets/images/purpleGB.png
---

# PurpleGB: Construindo um Emulador de Game Boy em C

**Um mergulho de um dia na arquitetura de 8 bits, renderização por scanline e nos bugs que só aparecem quando você constrói tudo do zero.**

---

Se você passou algum tempo na cena de emulação, provavelmente me conhece como MrPurple666. Eu contribuí para o Eden, um emulador de Nintendo Switch. Modifiquei drivers Turnip para Mesa3D no Android, empurrando a compatibilidade de GPUs mobile para frente. Já debbuguei shaders, tracei syscalls e encarei mais dumps hexadecimais do que gostaria de admitir.

Mas tem uma coisa que eu nunca tinha feito: construir um emulador do zero.

Eu já fiz patches, perfilei gargalos e corrigi bugs críticos nos projetos de outras pessoas. Mas a fatia vertical completa, desde CPU e barramento de memória até PPU, APU e mapeadores de cartucho, desde ligar a energia até o primeiro pixel, sempre foi arquitetura de outra pessoa. Eu sabia como emuladores funcionavam porque vivi dentro deles por anos. Só nunca tinha construído um.

Essa lacuna me incomodava. Você só pode ler tantos diagramas de timing de PPU e revisões do Pan Docs antes de começar a se perguntar se realmente *entende*, ou se é apenas muito bom em reconhecer padrões nas implementações dos outros.

Então eu resolvi descobrir.

---

## Por que o Game Boy?

O Game Boy é o teste de honestidade perfeito. Pequeno o bastante para uma pessoa terminar numa sessão focada. Cerca de 2600 linhas de C em 8 arquivos fonte, um Makefile de 3 linhas. Complexo o bastante para não dar pra fingir. Cada opcode errado, cada ciclo a mais ou a menos na PPU, cada bit invertido no registrador do joypad aparece na tela imediatamente, em tiles distorcidos, áudio silencioso e crashes que não fazem sentido até que, de repente, fazem.

Não tem OS pra culpar. Se o logo da Nintendo não aparece, é porque *a sua* decodificação de nibble está errada. Se o Tetris trava na tela de título, é porque *o seu* timing de interrupção está fora. O Game Boy não liga pro seu currículo.

Foi por isso que eu escolhi ele. Não apesar da simplicidade, mas por causa dela.

---

## Visão Geral da Arquitetura

O PurpleGB é uma implementação de um dia em C99 com SDL3 para vídeo e input. A arquitetura é deliberadamente plana:

| Módulo | Responsabilidade | Linhas |
|--------|-----------------|--------|
| `cpu.c` | Execução de instruções SM83, 512 opcodes, manipulação de flags | ~600 |
| `memory.c` | Barramento de memória, carregamento de cartuchos, banking MBC1/3/5, dispatch de I/O | ~400 |
| `ppu.c` | Renderizador por scanline: background, window, sprites, LCDC/STAT | ~350 |
| `interrupt.c` | Dispatch de IE/IF/IME, VBlank/STAT/Timer/Serial/Joypad | ~80 |
| `timer.c` | DIV/TIMA/TMA/TAC com clock-select | ~100 |
| `joypad.c` | Registrador P1, mapeamento de teclas SDL3 | ~60 |
| `apu.c` | Esqueleto de áudio de 4 canais (ainda não funcional) | ~200 |
| `main.c` | Janela SDL3, loop de eventos, logo de boot, drag-and-drop, bandeja, FPS | ~700 |

A CPU é um interpretador cycle-stepped. A PPU renderiza por scanline, não por pixel. O banking de memória suporta mapeadores MBC1, MBC3 e MBC5. O áudio está em stub — os registradores existem, o stream SDL3 de áudio é inicializado, mas o pipeline de síntese ainda não está conectado.

---

## Os Bugs que Importam

### A Tela Listrada: Endereçamento de Tiles

A primeira saída gráfica ficou assim:

![Primeira saída gráfica — listras pretas e brancas](/assets/images/purpleGB/1.png)

*Listras onde deveria estar o Tetris. A PPU está viva, o LCD está ligado, o cartucho está rodando. Mas algo está profundamente errado com o fetching de tiles.*

O problema era o bit 4 do LCDC. Ele controla se os tiles de background/window usam endereçamento sem sinal (`$8000-$8FFF`) ou com sinal (`$8800-$97FF`). Eu tinha a lógica invertida. Cada índice de tile estava sendo lido do banco errado, produzindo as listras alternadas.

Forçar o modo unsigned manualmente produziu isso:

![Escala de cinza mas ainda corrompido](/assets/images/purpleGB/1.2.jpg)

*O pipeline de paleta funciona. Escala de cinza DMG em vez de preto e branco puro. Mas os tiles ainda estão errados. Progresso, eu acho.*

### A Tela Branca do Quase

Depois de corrigir o endereçamento de tiles, tudo quebrou. A tela ficou branca. Não era branco de crash. A janela estava aberta, o loop de eventos rodando, o cartucho carregado. Só branco. Todo pixel da mesma cor.

![Tela branca — nada renderizando](/assets/images/purpleGB/2.png)

*A PPU está rodando, o LCD está ligado, o framebuffer está sendo preenchido de branco. Todo frame, para sempre.*

O bug estava na sincronização de registradores entre `memory.c` e `ppu.c`. Os registradores de I/O do Game Boy ficam em `$FF00-$FF7F`. A PPU precisa lê-los a cada ciclo para saber o que renderizar. Eu tinha cópias em cache na struct da PPU que nunca eram atualizadas quando a CPU escrevia na memória de I/O. A PPU renderizava com valores desatualizados — LCDC ainda no padrão de power-on, SCX e SCY em zero, o ponteiro do background map apontando para lugar nenhum.

Quatro horas pra encontrar. Quatro horas de printf debugging, verificando cada escrita de memória, confirmando que a CPU estava fazendo exatamente o que deveria. A CPU estava bem, o barramento de memória estava bem, a PPU estava bem. A conexão entre elas estava quebrada.

Esse é o tipo de bug que você só pega construindo a coisa você mesmo.

### Fragmentos de Copyright

Depois de corrigir a sincronização de registradores, algo mágico aconteceu. A tela não estava mais branca. Ela era... quase legível. Pequenos fragmentos de texto espalhados pela borda superior, pareciam aspas, apóstrofos e pedaços de letras.

![Primeiros tiles visíveis — fragmentos do copyright do Tetris](/assets/images/purpleGB/3.png)

*Os primeiros pixels reconhecíveis do cartucho. Esses são pedaços de "© 1989". O endereçamento do background map ainda está errado, mas os dados dos tiles finalmente estão chegando à tela.*

Aqueles fragmentos eram a tela de copyright do Tetris. O jogo estava tentando mostrar "© 1989 Nintendo", e eu estava recebendo aspas, apóstrofos e pedaços de 9. Como tentar ler um jornal através de vidro fosco, mas era algo. O decodificador de tiles funcionava. A camada de background renderizava. Os registradores de scroll estavam perto.

Aí a tela completa apareceu, e eu descobri um novo problema: ela estava presa em um loop. A tela de copyright renderizava completamente, depois reiniciava de cima, depois renderizava de novo, depois reiniciava. A interrupção de VBlank estava disparando, mas `LY` (o registrador de scanline atual) não estava incrementando corretamente pelo período de VBlank. A PPU achava que ainda estava na scanline 144, então ficava disparando VBlank, o que resetava o contador de frame, o que disparava VBlank de novo.

![Tela de copyright — presa em loop de reinício](/assets/images/purpleGB/4.png)

*A tela de copyright completa, finalmente visível. Mas ela está se reescrevendo a cada poucos frames, presa em um loop porque LY nunca avança além do limite de VBlank.*

A correção foi um único `break` fora do lugar no ciclo de modo da PPU. Uma palavra-chave. Várias horas.

### As Cores Estavam Mentindo

O Tetris chegou na tela de título. Isso deveria ser um momento de celebração. Em vez disso, eu recebi isso:

![Tela de título com cores erradas](/assets/images/purpleGB/5.png)

*A tela de título. Reconhecível, mas errada. O logo "TETRIS" está lá, mas as cores estão invertidas. Preto onde deveria ser cinza claro, cinza claro onde deveria ser preto.*

O problema era o registrador BGP — Background Palette em `$FF47`, um único byte que mapeia as quatro cores DMG (0 a 3) para quatro tons de cinza. O Tetris estava escrevendo o valor correto no BGP, mas meu emulador estava interceptando essa escrita e corrompendo-a.

Por quê? Por causa de um fallthrough de `switch` no meu handler de escrita de I/O. Eu tinha agrupado vários registradores da PPU juntos:

```c
// QUEBRADO: todos esses caem no handler de DMA
case 0x41: case 0x42: case 0x43:
case 0x45: case 0x47: case 0x48: case 0x49:
case 0x46: {
    m->io[0x46] = v;  // Toda escrita vira um gatilho de DMA!
    // ... transferência DMA acontece aqui ...
    break;
}
```

`0x46` é o registrador de OAM DMA, `0x41` é STAT e `0x47` é BGP. Toda vez que o jogo escrevia em STAT, SCX, SCY, LYC, BGP, OBP0 ou OBP1, meu emulador tratava como uma requisição de transferência DMA. O DMA disparava, corrompendo a memória OAM com dados de qualquer endereço que estivesse no byte "fonte" (que na verdade era o valor da paleta, ou a posição de scroll, ou as flags de STAT). Esses dados OAM corrompidos alimentavam a renderização de sprites, que corrompia outra memória, que eventualmente fazia o jogo pular para endereços inválidos.

Um `break` faltando. Oito registradores corrompidos. Incontáveis horas debugando sintomas que não tinham nada a ver com a causa raiz.

### O Logo da Nintendo Que Não Estava Lá

A boot ROM do Game Boy é famosa por sua sequência de startup. Ela mostra o logo da Nintendo, desce ele na tela, toca o som de boot e entrega o controle pro cartucho. Como eu pulo a boot ROM (começando a execução em `$0100` em vez de `$0000`), eu precisei replicar a exibição do logo em software.

O logo é codificado no cabeçalho do cartucho em `$0104-$0133`, 48 bytes que representam o logo da Nintendo como um mapa de tiles comprimido. A boot ROM decodifica cada nibble através de uma sequência engenhosa de rotações `RL B` / `RLA`, expandindo 4 bits em padrões de 8 bits de linha de tile. Eu simulei esse algoritmo, pré-computei a tabela completa de 384 bytes e incorporei como um array estático.

A primeira tentativa ficou assim.

![Primeira tentativa do boot logo — corrompido](/assets/images/purpleGB/6.png)

*Meu primeiro boot logo. O logo da Nintendo está quase lá. Dá pra ver a forma geral, os dois olhos circulares, as linhas curvas. Mas os padrões de bits estão errados. Um único comportamento de carry flag estava errado na minha expansão de nibble.*

Era perto. Frustrantemente perto, enlouquecedoramente perto. Eu conseguia ver o que deveria ser, mas cada tile tinha pixels errados. O problema era o comportamento exato da flag de carry através de quatro iterações de `RL B` / `RLA`. A documentação do Z80 é ambígua em alguns casos extremos, então eu tinha chutado errado. Quando finalmente combinei o comportamento do hardware exatamente — traçando cada passo de rotação um por um e comparando contra dumps conhecidos da boot ROM. O logo decodificou perfeitamente.

Mas ainda não aparecia na tela.

Porque o Tetris, como a maioria dos cartuchos, limpa a VRAM durante sua rotina de inicialização. Meu logo lindamente decodificado estava sendo apagado antes do primeiro frame ser apresentado. A correção foi um estado dedicado de splash de boot que renderiza o logo por 90 frames (~1,5 segundos) antes de iniciar a CPU:

```c
g->boot_frames = 90;
if (g->boot_frames > 0) {
    ppu_render_frame(&g->ppu, &g->mem);
    g->boot_frames--;
}
```

Simples, e óbvio em retrospectiva. Mas eu passei um tempo convencido de que minha decodificação do logo ainda estava errada, antes de perceber que o cartucho estava deletando ele.

### A Correção Que Corrigiu Tudo

O fallthrough do dispatch de I/O não estava só quebrando as cores. Estava quebrando tudo. Toda escrita em um registrador da PPU estava corrompendo a OAM, que estava corrompendo sprites, que estava corrompendo o estado interno do jogo, que causava crashes, travamentos e glitches gráficos que pareciam completamente sem relação.

Quando finalmente separei os casos — dando a STAT, SCX, SCY, LYC, BGP, OBP0 e OBP1 seus próprios handlers adequados. Tudo se encaixou de uma vez.

![Tela de título do Tetris — correta](/assets/images/purpleGB/7.png)

*A tela de título, finalmente correta. O logo TETRIS na escala de cinza DMG adequada, o padrão de background intacto, a camada de window (aquela barra de status embaixo) renderizando no lugar certo.*

E aí, o jogo:

![Tetris em jogo — correto](/assets/images/purpleGB/8.png)

*Tetris, totalmente jogável. Tiles de background, camada de window, sprites dos blocos caindo, display de pontuação. Tudo renderizando corretamente depois do commit `fix: restore LCD register writes and DMA dispatch`.*

Eu fiquei olhando pra essa tela por um bom tempo. Não porque fosse impressionante. É só Tetris, rodando a 160 por 144 de resolução, com quatro tons de cinza. Mas porque eu sabia exatamente o que cada pixel representava. Eu sabia o opcode que buscou o índice do tile. Eu sabia o endereço de memória onde aquele tile morava. Eu sabia a manipulação de bits que transformava dois bytes de dados de tile em oito pixels. Eu sabia a lookup de paleta que mapeava índice de cor 2 para cinza claro.

Eu sabia porque eu tinha construído tudo, desde main() até o último pixel.

---

## Status Atual

O PurpleGB é uma implementação de um dia. Não é um emulador pra mostrar. Não tem save states, rewind, netplay, ou um pipeline chique de shaders. O que ele tem:

| Componente | Status |
|-----------|--------|
| CPU SM83 (512 opcodes) | ⚠️ Parcial |
| PPU por scanline (BG/Window/Sprites) | ⚠️ Parcial |
| Timer (DIV/TIMA/TMA/TAC) | ⚠️ Parcial |
| Joypad (registrador P1, input SDL3) | ⚠️ Parcial |
| Banking de cartuchos MBC1/3/5 | ⚠️ Parcial |
| Controlador de interrupções (VBlank/STAT/Timer/Serial/Joypad) | ⚠️ Parcial |
| Logo de boot / caminho Bootix | ⚠️ Parcial |
| Carregamento de ROM por drag-and-drop | ⚠️ Parcial |
| Overlay de pausa com ESC | ⚠️ Parcial |
| Ícone na bandeja do sistema | ⚠️ Parcial |
| Contador de FPS | ⚠️ Parcial |
| APU (áudio de 4 canais) | ❌ Stub — stream SDL3 inicializado, síntese não conectada |
| Saída de áudio | ❌ Não funcional |

**Compatibilidade de ROMs:** Tetris roda perfeitamente. Esse é o único título verificado neste estágio. Pokémon Green e outros jogos ainda não foram validados.

---

## O Que Esse Projeto Realmente É

O PurpleGB nunca vai competir com SameBoy ou BGB. Seu timing de PPU é aproximado, não cycle-accurate. Seu áudio é silencioso. Ele vai crashar em ROMs de teste de casos extremos. Ele foi construído numa única sessão focada, não ao longo de meses de testes cuidadosos de hardware.

Mas ele me ensinou algo que aqueles emuladores não podiam: como é sentar na frente de uma tela branca às 2 da manhã, sabendo que em algum lugar em 2600 linhas de C, um único bit está errado e você tem que encontrar ele. Como é ver aspas e apóstrofos espalhados numa tela corrompida, e reconhecê-los como fragmentos de uma mensagem de copyright que você ainda não mereceu. Como é finalmente ver o logo da Nintendo. Não porque você carregou uma ROM, mas porque o seu algoritmo de expansão de nibble, traçado através de quatro iterações de RL B e RLA, produziu o padrão de bits certo.

Foi por isso que eu construí isso. Não pra adicionar mais um emulador no mundo, mas pra fechar uma lacuna que eu carregava há anos.

---

## Atualização: O Que Mudou Depois Deste Post

Os commits mais recentes do PurpleGB não mudaram a ideia do artigo; eles substituíram alguns atalhos que ajudavam durante o debug, mas estavam errados como comportamento de emulador.

- `c304221` troca o caminho de splash de boot logo feito à mão por uma boot ROM DMG do Bootix embutida. Agora o emulador começa em `$0000`, roda a boot ROM até ela se desativar por `$FF50`, e só então expõe o código do cartucho em `$0100`.
- `d8ca59e` remove dívida de limpeza: o modelo morto de APU, o caminho falso de boot logo, o tracing persistente de opcode indefinido, o estado inalcançável de DMA adiado, o scaffolding incompleto de bandeja, código duplicado de carregamento de ROM, o include SDL não usado na PPU e campos de estado não usados.
- `1e695fc` corrige o tratamento de cores da PPU mantendo os índices crus de cor de BG/window antes do mapeamento pela paleta. Isso importa para prioridade de sprites, porque no DMG a prioridade checa se o índice de cor do BG é zero, não se o tom final de cinza ficou branco.
- `d6a6a2a` corrige o bug de dispatch de I/O descrito acima: `$FF41-$FF45`, `$FF47-$FF49` e `$FF4A-$FF4B` são registradores normais de LCD/status/scroll/paleta/window; só `$FF46` é OAM DMA.

Então a história de debug acima continua como registro do processo, mas o emulador atual não depende mais do workaround temporário de splash screen nem do handler agrupado de I/O que causava a corrupção de cores e DMA.

---

## Agradecimentos

- [Pan Docs](https://gbdev.io/pandocs/) — a referência técnica canônica do Game Boy. Cada registrador, cada comportamento de flag, cada quirk da PPU documentado num só lugar.
- [SameBoy](https://sameboy.github.io/) e [BGB](https://bgb.bircd.org/) — emuladores de referência que eu comparei quando minha saída estava errada e eu precisava saber como "certo" deveria parecer.
- [Bootix](https://github.com/Ashiepaws/Bootix) — O disassembly da boot ROM em `references/Bootix/bootix_dmg.asm` que me permitiu verificar meu algoritmo de expansão de nibble contra o comportamento real do hardware.
- [SDL3](https://libsdl.org/) — O framework que cuida de janela, input e stream de áudio, pra eu poder focar na emulação em si.
- A comunidade de emulação do Game Boy — por décadas de engenharia reversa, ROMs de teste e conhecimento compartilhado que torna projetos como esse possíveis em um único dia.

---

*Fonte: [github.com/MrPurple666/PurpleGB](https://github.com/MrPurple666/PurpleGB)*
