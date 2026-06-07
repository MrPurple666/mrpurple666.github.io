---
title: "Engenharia Reversa do fTPM"
permalink: /projects/ftpm-pt/
---

# Engenharia Reversa de Driver UEFI SMM para Firmware TPM (fTPM)

## Sumário
- [Contexto Técnico e Inicialização](#contexto-técnico-e-inicialização)
- [Fluxo de Execução Integrado](#fluxo-de-execução-integrado)
- [Variáveis Globais e Estruturas Críticas](#variáveis-globais-e-estruturas-críticas)
  - [FTPM_GLOBAL_DATA](#ftpm_global_data-39-bytes-smram)
  - [TPM2_CRB_CONTROL_AREA](#tpm2_crb_control_area-48-bytes)
  - [TPM2_ACPI_TABLE](#tpm2_acpi_table-52-bytes)
- [Handlers SW-SMI e Interação com o Sistema Operacional](#handlers-sw-smi-e-interação-com-o-sistema-operacional)
- [Observações Técnicas](#observações-técnicas)

---

## Contexto Técnico e Inicialização

Recentemente conduzi a análise de um driver UEFI SMM que integra um Firmware TPM (fTPM) baseado em ARM TrustZone. O repositório é privado, pois inclui processos internos, handlers e buffers críticos, mas o estudo permite compreender em detalhes a comunicação entre firmware, SMRAM e TPM.

Este driver atua como uma ponte entre o firmware UEFI e uma *Trusted Application* no TrustZone. Ele não implementa a lógica interna do TPM, mas garante que todas as operações críticas requisitadas pelo Sistema Operacional sejam processadas de forma segura e confiável.

O driver executa no System Management Mode (SMM), o nível mais privilegiado da CPU, operando de forma invisível ao sistema operacional. A inicialização ocorre em duas fases:

* **DXE Phase:** `InitializeSmmBase` salva handles globais essenciais, localiza protocolos necessários e verifica a existência de hardware fTPM via MMIO no endereço `0xE00D0000`.
* **SMM Phase:** `FtpmSmmInit` registra os handlers de *Software System Management Interrupts* (SW-SMI), instala a tabela ACPI TPM2 e configura os buffers de comando/resposta (CRB).

A persistência do driver é mantida através de `setjmp` e `longjmp`, permitindo residir em memória mesmo diante de falhas internas.

---

## Fluxo de Execução Integrado

```text
[DXE Phase] InitializeSmmBase
    │
    ├─> Salva handles globais (ImageHandle, SystemTable, Boot Services)
    ├─> Localiza protocolo EFI_SMM_BASE2 → gSmmBase2
    ├─> Consulta tamanho do fTPM e aloca buffer SMM
    └─> Verifica hardware fTPM via MMIO

[SMM Phase] FtpmSmmInit
    │
    ├─> CheckFtpmHardware — aborta se hardware ausente
    │
    ├─> SetupAcpiTable
    │      ├─> Recupera blob de capacidades do fTPM
    │      ├─> Aloca gFtpmData (39 bytes na SMRAM)
    │      ├─> Corrige ponteiros na estrutura ACPI ("TNVF")
    │      └─> Instala tabela ACPI TPM2 (52 bytes)
    │
    ├─> Registra Handlers SW-SMI
    │      ├─> PhysicalPresenceSmiHandler — manipula TrEEPhysicalPresence
    │      └─> MemoryOverwriteSmiHandler — manipula MemoryOverwriteRequestControl
    │
    └─> InstallTpmProtocol
           ├─> Preenche CRB (48 bytes)
           ├─> Define CommandAddress / ResponseAddress = CRB_base + 0x80
           └─> Instala tabela ACPI final
```

---

## Variáveis Globais e Estruturas Críticas

| Variável | Tipo | Descrição |
| :--- | :--- | :--- |
| `gImageHandle` | `EFI_HANDLE` | Handle da imagem do driver |
| `gSystemTable` | `EFI_SYSTEM_TABLE*` | Ponteiro para a tabela central do sistema |
| `gBS_local` | `EFI_BOOT_SERVICES*` | Boot services de escopo local |
| `gSmst` | `EFI_SMM_SYSTEM_TABLE2*` | Ponteiro para a Tabela SMM |
| `gSmmBase2` | `EFI_SMM_BASE2_PROTOCOL*` | Protocolo base da infraestrutura SMM |
| `gSmmVariable` | `EFI_SMM_VARIABLE_PROTOCOL*` | Protocolo para acesso e controle de variáveis SMM |
| `gInSmm` | `BOOLEAN` | Flag indicando se a execução está dentro da SMRAM |
| `gAllocatedPool` | `VOID*` | Buffer de memória pré-alocado dedicado ao fTPM |
| `gFtpmDeviceFamily` | `UINT32` | Identificador numérico da família do dispositivo fTPM |
| `gFtpmData` | `FTPM_GLOBAL_DATA*` | Dados compartilhados para controle de PP/MOR |
| `gFtpmCrbSnapshot` | `TPM2_CRB_CONTROL_AREA` | Snapshot do estado do CRB |
| `gTpm2AcpiTable` | `TPM2_ACPI_TABLE` | Tabela ACPI TPM2 com tamanho fixo de 52 bytes |
| `gDriverStatus` | `EFI_STATUS` | Status e código de retorno interno do driver |
| `gJmpBuf` | `jmp_buf` | Armazena contexto CPU para setjmp/longjmp |

### FTPM_GLOBAL_DATA (39 bytes, SMRAM)

```c
#pragma pack(1)
typedef struct {
    UINT8   PpSwSmiValue;
    UINT32  PpRequest;
    UINT32  PpRequestParameter;
    UINT32  PpLastRequest;
    UINT32  PpPendingResponseFlags;
    UINT32  PpOperationResponse;
    UINT8   MorSwSmiValue;
    UINT32  MorOperation;
    UINT8   MorSavedByte;
    UINT8   Reserved[3];
    UINT32  MorStatus;
} FTPM_GLOBAL_DATA;
#pragma pack()
```

### TPM2_CRB_CONTROL_AREA (48 bytes)

```c
#pragma pack(1)
typedef struct {
    UINT32  ReqCancel;
    UINT32  ReqComplete;
    UINT32  Status;
    UINT32  Reserved;
    UINT64  CommandSize;
    UINT64  CommandAddress;
    UINT64  ResponseSize;
    UINT64  ResponseAddress;
} TPM2_CRB_CONTROL_AREA;
#pragma pack()
```

### TPM2_ACPI_TABLE (52 bytes)

A tabela ACPI TPM2 é instalada em dois estágios: placeholder e final, com `AddressOfControlArea` preenchido dinamicamente apontando para o CRB.

---

## Handlers SW-SMI e Interação com o Sistema Operacional

O SO interage com SMM via variáveis controladas na NVRAM, acionando Software SMIs. Os handlers principais são:

| Handler | Variável em NVRAM | Operações Mapeadas |
| :--- | :--- | :--- |
| `PhysicalPresenceSmiHandler` | `TrEEPhysicalPresence` | `PP_REQUEST_CLEAR_ACTIVATE`, `PP_REQUEST_DEACTIVATE_DISABLE`, `PP_REQUEST_SPECIAL` |
| `MemoryOverwriteSmiHandler` | `MemoryOverwriteRequestControl` (MOR) | `MOR_OP_DISABLE`, `MOR_OP_CLEAR` |

Fluxo dinâmico durante comunicação:

```text
[SO] envia comando → escreve na variável gFtpmData
          │
          v
[SMM] dispara SW-SMI
          │
          ├─> PhysicalPresenceSmiHandler → atualiza TrEEPhysicalPresence
          │
          ├─> MemoryOverwriteSmiHandler → atualiza MemoryOverwriteRequestControl
          │
          v
[Tabela ACPI TPM2] → referencia CRB_base
          │
          v
[CRB Buffer] → operação de comando/resposta compartilhada
```

---

## Observações Técnicas

1. **Limitações de Endereçamento:** gFtpmData é truncado para 32 bits na ACPI.
2. **Controle Irrestrito:** execução em SMM permite controle total da memória física do sistema.
3. **Segurança e Mitigações:** buffers pequenos validados mitigam ausência de stack canaries.
4. **Sigilo de Informação:** código e processos internos permanecem privados, incluindo despacho de handlers e formatos de buffers críticos.
