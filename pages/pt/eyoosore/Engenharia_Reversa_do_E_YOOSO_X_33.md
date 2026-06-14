---
title: "Engenharia Reversa do Mouse E-YOOSO X-33"
permalink: /pt/projects/eyoosore/
layout: single
lang: pt-BR
locale: pt-BR
project: true
translation_key: project-eyoosore
alternate_url: /projects/eyoosore/
header:
  image: /assets/images/eyoosore1.png
---

# Engenharia Reversa do Mouse E-YOOSO X-33: Do Driver Windows ao Aplicativo Linux

**Uma jornada por USB HID, Ghidra e protocolos indocumentados**

---

Comprei um mouse gamer programável E-YOOSO X-33. É um hardware respeitável. Sensor PMW3335, 16 botões programáveis, iluminação RGB, 5 níveis de DPI, gravação de macros, o pacote completo. Mas como a maioria desses periféricos fabricados na China, o software de configuração só roda no Windows.

Eu uso Linux (CachyOS, derivado do Arch). Então fiz o que qualquer usuário Linux que se preze faria: decidi construir uma ferramenta de configuração nativa do zero. O que se seguiu foi um mergulho profundo em USB HID, engenharia reversa de executáveis Windows PE, protocolos de EEPROM e desenvolvimento com GTK4.

Este post documenta a jornada.

---

## O Alvo

O software oficial do Windows é distribuído como um instalador Inno Setup: `E-YOOSO X-33 Setup v3.1 20231115(2).exe`. Dentro:

- `OemDrv.exe`: o aplicativo principal de configuração
- `Lowerdev.dll`: uma fina camada de comunicação HID
- `InitSetup.dll`, `MenuEx.dll`: bibliotecas de interface
- `Cfg.ini`: armazenamento de configuração
- `Text/`: tabelas de strings multilíngues (Inglês, Chinês, Russo, Português)

O `Cfg.ini` deu as primeiras pistas:

```ini
VID=0x25A7
PID=0xFA08
PID2=0xFA07
Sensor=0x3335
DM=5
DPI=1000,2000,4000,8000,16000
K1_1=0x02,0x31,0x00,0x01
K2_1=0x02,0x32,0x00,0x02
...
K16_1=0x01,0x13,0x00,0x07
```

Dois PIDs: `0xFA08` para modo com fio, `0xFA07` para o receptor wireless 2.4GHz. As entradas `K{n}_{m}` codificam 16 mapeamentos de botão no formato `tipo, codigo_da_tecla, parametro_extra, indice_hardware`. O sensor é PMW3335.

## Extraindo o Driver do Windows

Desempacotar instaladores Inno Setup no Linux é direto com `innoextract`:

```bash
innoextract "E-YOOSO X-33 Setup v3.1 20231115(2).exe"
```

Isso me deu a árvore completa da aplicação: DLLs, o executável principal, imagens de skin e os arquivos de texto. Converti os arquivos XML UTF-16 LE para texto legível e tinha um mapa completo de cada string de UI que o app Windows usa.

## Primeiro Contato: Enumeração USB

O mouse aparece no barramento USB como dois dispositivos diferentes dependendo do modo de conexão:

```
Com fio:    ID 25a7:fa08  "2.4G Dual Mode Mouse"
Wireless:   ID 25a7:fa07  "Compx 2.4G Wireless Receiver"
```

Ambos expõem duas interfaces HID:

- **Interface 0**: mouse HID padrão (5 botões + scroll + scroll horizontal)
- **Interface 1**: teclado HID + configuração vendor-specific

O descritor de relatório HID da interface de configuração revelou os canais de comunicação:

```
Report 0x01: INPUT  (7B)    modificadores de teclado + códigos de tecla
Report 0x02: INPUT  (7B)    status vendor / bateria
Report 0x03: INPUT  (1B)    system control
Report 0x05: INPUT  (2B)    consumer control
Report 0x06: FEATURE(7B)    config geral (perfil / polling)
Report 0x08: FEATURE(16B)   dados vendor (inicialmente achei que eram botões)
Report 0x09: INPUT  (16B)   input vendor
```

Minha primeira suposição foi que `SET_REPORT` no report `0x08` configuraria os mapeamentos de botão. Passei dois dias testando cada combinação de tamanhos de buffer, IDs de report e tipos de transferência. As escritas sempre retornavam sucesso no nível USB. Mas os valores nunca persistiam. O readback sempre retornava zeros.

## O `@property` Que Me Custou Horas

Durante o desenvolvimento, acidentalmente removi o decorador `@property` do método `is_connected` durante um refactor. Isso fez com que `self._mouse.is_connected` retornasse um objeto de método vinculado em vez de `True`/`False`. O timer de `_scan_devices` nunca detectava o mouse, mostrando "Desconectado" mesmo com o dispositivo plugado e o CLI provando que a comunicação funcionava.

Um decorador faltando. Horas de debugging. Clássico.

## Lowerdev.dll: Um Beco Sem Saída

Desmontei a `Lowerdev.dll` e descobri que é puramente um wrapper fino:

```c
// São literalmente chamadas repassadas
SetFeature       -> HidD_SetFeature
GetFeature       -> HidD_GetFeature
SetOutputReport  -> HidD_SetOutputReport
GetInputReport   -> HidD_GetInputReport
```

Zero lógica de protocolo. A mágica real está no `OemDrv.exe`.

## Ghidra ao Resgate

Carreguei o `OemDrv.exe` no Ghidra. O binário é uma aplicação GUI Win32 compilada em Delphi. É enorme, com fluxo de controle spaghetti típico de código orientado a eventos em VCL.
A primeira descoberta veio do rastreamento do carregador de wrappers HID:

```c
void LoadHidWrappers(HMODULE hLowerdev) {
    g_HidApi.OpenHidDevice   = GetProcAddress(hLowerdev, "OpenHidDevice");
    g_HidApi.SetFeature      = GetProcAddress(hLowerdev, "SetFeature");
    g_HidApi.GetFeature      = GetProcAddress(hLowerdev, "GetFeature");
    g_HidApi.SetOutputReport = GetProcAddress(hLowerdev, "SetOutputReport");
    g_HidApi.GetInputReport  = GetProcAddress(hLowerdev, "GetInputReport");
}
```

A aplicação abre **dois** handles HID, não um:

```c
// Handle de comando
HANDLE hCmd = OpenHidDevice(VID, PID, usagePage=0xFF02, usage=2);

// Handle de notificação
HANDLE hNotify = OpenHidDevice(VID, PID, usagePage=0xFF01, usage=0);
```

Isso foi crucial. `0xFF02` e `0xFF01` correspondem às duas coleções vendor-specific no descritor de relatório HID. O driver Windows as abre por *usage page*, não por número de interface. No Linux, mapeei o nó de comando para `/dev/hidraw3` no modo wireless e `/dev/hidraw5` no modo com fio. É o nó hidraw que responde a um `GET_FEATURE` no report `0x06`.

## O Protocolo Real: Comandos EEPROM

A próxima descoberta veio da descompilação do helper de pacotes de comando:

```c
void SendCommandPacket(HANDLE hCmd, uint8_t cmd, int body...) {
    uint8_t buf[17];
    buf[0] = 0x08;  // ID do report
    buf[1] = cmd;   // ID do comando
    // ... bytes do corpo ...
    buf[16] = 0x55 - sum(buf[0..15]);  // checksum
    SetFeature(hCmd, buf, 17);
}
```

A comunicação real é um **protocolo de comandos de 17 bytes**, não leituras feature HID padrão:

| Byte | Propósito |
|------|-----------|
| 0 | ID do Report (sempre `0x08`) |
| 1 | ID do Comando |
| 2-15 | Payload específico do comando |
| 16 | Checksum (`0x55 - soma(bytes[0..15])`) |

Comandos que identifiquei:

| Comando | Função |
|---------|--------|
| `0x03` | GetConnectStatus (preflight antes de qualquer escrita) |
| `0x07` | WriteEEPROM |
| `0x08` | ReadEEPROM |
| `0x09` | Commit / Aplicar |

Os comandos `0x01` (handshake), `0x02` e `0x04` também existem mas ainda não os mapeei completamente.

As respostas vêm como pacotes prefixados com `0x09` no mesmo nó hidraw:

```
09070000020253000000000000000000ee
|  | |  |
|  | |  +-- checksum
|  | +----- payload ecoado
|  +------- byte de status
+---------- marcador de resposta (0x09)
```

O insight crucial: cada escrita EEPROM é confirmada por um pacote de resposta com o mesmo formato da requisição.

## Mapa de Memória EEPROM

Da função principal "Apply" no `OemDrv.exe`, reconstruí o layout da EEPROM:

| Endereço | Tamanho | Conteúdo |
|----------|---------|----------|
| `0x0000` | 2B | Código da polling rate + checksum |
| `0x000C` | 4B x 5 | Tabela DPI (5 níveis, 4 bytes cada) |
| `0x002C` | 4B x 5 | Tabela de cores DPI (R,G,B + checksum por nível) |
| `0x0060` | 4B x 16 | Matriz de teclas (16 botões, 4 bytes cada) |
| `0x0100`+ | variável | Blobs extras por tecla (combos de teclado) |
| `0x0300`+ | variável | Armazenamento de macros (16 slots) |

A tabela DPI usa uma lookup table específica do sensor. Extraí a tabela completa de códigos DPI do sensor `0x3335` do `OemDrv.exe`:

```python
SENSOR_3335_DPI_TO_CODE = {
    100: 0x00, 200: 0x02, 300: 0x03, 400: 0x04, 500: 0x05,
    ...
    4000: 0x2F, ..., 16000: 0xBD,
}
```

O formato de entrada da matriz de teclas codifica a função:

```c
// Entradas de função do mouse
LeftButton    -> 0x000101
RightButton   -> 0x000201
FireKey       -> 0x000401
SniperButton  -> 0x000801

// Tecla simples de teclado
SingleKey     -> 0x000005  // mais um blob de 7 bytes em (índice+8)*0x20
```

## O Problema do Wake

Mouses wireless entram em repouso para economizar bateria. Quando isso acontece, o receptor aceita os pacotes de comando, mas não envia ACKs. Descobri isso na prática: escritas que funcionavam logo após mover o mouse falhavam quando ele ficava parado por alguns minutos.

A correção foi abrir o nó de input do mouse, aquele que retorna `Broken pipe` para *feature reports* mas transmite *input reports* quando o mouse se move, e ler dele antes de cada operação de escrita. Isso gera tráfego USB suficiente para acordar o rádio do mouse:

```python
def _wake_mouse(self):
    """Lê do nó de input para acordar o mouse wireless."""
    for _ in range(3):
        try:
            data = os.read(self._notify_fd, 64)
            if data:
                return True
        except BlockingIOError:
            pass
        time.sleep(0.1)
    return True
```

## Construindo a Aplicação Linux

A ferramenta Linux é uma aplicação GTK4 / libadwaita em Python. Arquitetura:

```
eyooso_mouse/
├── mouse_protocol.py    # Parsing de descritores HID, protocolo EEPROM,
│                        # transporte hidraw ioctl, tabelas DPI/sensor
├── config_manager.py    # Parser de Cfg.ini, save/load de perfis JSON
├── main.py              # AdwApplicationWindow GTK4 com navegação em abas
└── ui/
    ├── main_page.py     # Visual interativo do mouse (renderização Cairo),
    │                    # painel de mapeamento de botões
    ├── advanced_page.py # Editor de DPI (5 níveis + amostras de cor),
    │                    # polling rate, sensibilidade, scroll
    ├── lighting_page.py # Modos LED (fixo/respiração/neon/streaming/off),
    │                    # brilho, velocidade, seletor de cor
    └── macro_page.py    # Gravação, edição, import/export de macros
```

Detalhes técnicos que valem mencionar:

- Usa a interface **hidraw** do kernel (`/dev/hidraw*`) via `fcntl.ioctl` com `HIDIOCGFEATURE` / `HIDIOCSFEATURE`, e não `pyusb` ou `libusb`. Esse caminho lida corretamente com os report IDs e retorna os pacotes completos.
- Implementa o algoritmo exato de checksum recuperado do binário descompilado: `0x55 - soma(bytes[0:15])`.
- Tem lógica de *retry* com *timeouts* adaptativos para a latência dos ACKs em modo wireless.
- Inclui 31 testes unitários com pytest cobrindo construção de pacotes, checksums, *lookup* da tabela DPI, codificação da matriz de botões e serialização de configuração.

## O Que Funciona (Junho 2026)

| Funcionalidade | Wireless (0xFA07) | Com fio (0xFA08) |
|----------------|--------------------|-------------------|
| Troca de perfil | ✅ | ✅ |
| Escrita da polling rate | ✅ | ✅ |
| Escrita da tabela DPI (0x3335) | ✅ | ✅ |
| Escrita de cores DPI | ✅ | ✅ |
| Escrita da matriz de teclas (fns mouse) | ✅ | ✅ |
| Teclas simples de teclado | ✅ | ✅ |
| Combos complexos de teclas | ❌ | ❌ |
| Gravação/escrita de macros | ❌ | ❌ |
| Escrita de modo LED | ❌ | ❌ |
| Leitura de bateria | ❌ | ❌ |

## O Que Ainda Falta

O protocolo de macros e combos complexos de teclas usa um formato de blob binário que eu ainda não decodifiquei completamente. O caminho de escrita de LED existe no `OemDrv.exe` descompilado, mas referencia endereços que eu ainda não mapeei. A leitura de bateria e da versão de firmware depende da thread do *notify handle*, que implementei só parcialmente.

O driver Windows também implementa um mecanismo de atualização de firmware (`ShowFWUpdate` no `Cfg.ini`) e um menu de DPI do botão sniper. Nenhum dos dois foi portado ainda.

## O Que Aprendi

Este projeto me ensinou mais sobre USB HID do que eu jamais esperava saber:

- **USB control transfers vs hidraw**: `pyusb` com *control transfers* funciona, mas os *ioctls* do hidraw são mais limpos no Linux. O kernel monta os pacotes completos de relatório HID para você.
- **Descritores de relatório HID dizem o que está declarado, não necessariamente o que funciona**: o dispositivo aceita `SET_REPORT` com IDs de report que não aparecem no descritor, desde que o formato do buffer corresponda ao transporte.
- **Receptores wireless armazenam estado em cache**: leituras de *feature report* retornam valores cacheados, não o estado real do dispositivo. A configuração do mouse vive na EEPROM interna e é encaminhada via wireless.
- **O descompilador do Ghidra é muito bom**: mesmo em binários compilados em Delphi com fluxo de controle VCL bagunçado, o pseudocódigo reconstruído foi preciso o bastante para portar a lógica de construção de pacotes diretamente para Python.
- **Dispositivos que entram em repouso são um problema real**: periféricos wireless hibernam agressivamente para economizar energia, e você precisa gerar tráfego USB no nó de input para acordá-los antes que os comandos de configuração sejam aceitos.

## Onde Estamos — E Por Que Ainda Não Funciona

Vou ser honesto: **este projeto ainda é experimental e não está totalmente funcional como ferramenta de uso diário.**

A ferramenta Linux compila, o código está sólido, o protocolo de pacotes está implementado corretamente e as escritas EEPROM são reconhecidas pelo mouse. Mesmo assim, ainda existe uma lacuna entre “o mouse confirma a escrita” e “o mouse realmente muda seu comportamento”.

Aqui está o que ainda está nos bloqueando:

1. **A condição de commit é frágil.** Às vezes o dispositivo aceita todas as escritas e o perfil muda instantaneamente. Em outras, principalmente quando o mouse está em modo wireless e não foi movido há algum tempo, a mesma sequência de pacotes é confirmada no nível USB, mas o firmware do mouse ignora as mudanças. O driver oficial do Windows parece ter uma sequência de *handshake* mais complexa, que ainda não consegui decodificar por completo. Muito provavelmente ela envolve o comando `0x01`, que envia bytes de desafio, e alguma etapa de validação da resposta.

2. **A máquina de estados interna do mouse não é documentada.** Depois de um ciclo de escrita e commit, o mouse entra brevemente em um “modo de programação”. Durante essa janela, o receptor para de encaminhar os relatórios de input do mouse para o host. Se você mover o mouse enquanto a configuração está sendo escrita, o cursor simplesmente para por cerca de 1,5 segundo. O driver do Windows leva esse timing em conta. A implementação Linux ainda não.

3. **Wireless e com fio se comportam de forma diferente.** Quando conectado via cabo USB (PID `0xFA08`), o mouse fala diretamente com o host e o nó de comando fica sempre responsivo. Via receptor 2.4GHz (PID `0xFA07`), o receptor age como proxy. Ele coloca comandos em buffer e só encaminha para o mouse quando o rádio acorda. Esse buffering introduz latência e, às vezes, perda total de pacotes. Eu já adicionei *retries*, mas ainda não tenho o protocolo de gerenciamento de buffer do receptor.

4. **Os protocolos de macro e LED ainda são caixas-pretas.** Eu já sei os endereços EEPROM onde os dados de macro vivem (a partir de `0x0300`) e consigo ver as funções que montam esses blobs no binário descompilado. O que ainda falta é entender o formato interno, isto é, como eventos de tecla, delays e controles de loop são codificados. O mesmo vale para as escritas do modo LED.

Não vou desistir, mas também não vou fingir que isso está perto de terminar. A distância entre “o mouse confirma a escrita” e “o botão realmente muda de função” não é pequena. Na prática, é a diferença entre conversar com o receptor e conversar com o firmware que roda dentro do mouse. Esse firmware não tem *datasheet*. Não tem SDK. A única documentação disponível, até agora, é um binário Delphi descompilado e muita tentativa e erro.

As peças que faltam:
- validação do *handshake*
- gerenciamento interno de buffer do receptor
- formato de codificação das macros

Esses são, individualmente, semanas de engenharia reversa. E qualquer uma delas pode ser exatamente a peça da qual todas as outras dependem.

**No próximo post, eu compartilho o que tiver conseguido avançar. Sem promessas de prazo. Isso aqui é hacking de fim de semana, não uma startup. Mas eu sou teimoso o bastante para continuar mexendo nisso.**

Até lá. Provavelmente não em breve. Mas uma hora sai.

---

*Escrito por um usuário Linux que cansou de reiniciar no Windows só para remapear um botão do mouse.*
