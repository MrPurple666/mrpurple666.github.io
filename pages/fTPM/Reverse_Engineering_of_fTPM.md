---
title: "Reverse Engineering of fTPM"
permalink: /projects/ftpm-en/
---

# Reverse Engineering of UEFI SMM Driver for Firmware TPM (fTPM)

## Table of Contents
- [Technical Context and Initialization](#technical-context-and-Initialization)
- [Integrated Execution Flow](#integrated-execution-flow)
- [Global Variables and Critical Structures](#Global-variables-and-critical-structures)
  - [FTPM_GLOBAL_DATA](#ftpm_global_data-39-bytes-smram)
  - [TPM2_CRB_CONTROL_AREA](#tpm2_crb_control_area-48-bytes)
  - [TPM2_ACPI_TABLE](#tpm2_acpi_table-52-bytes)
- [SW-SMI Handlers and Operating System Interaction](#sw-smi-handlers-and-operating-system-interaction)
- [Technical Observations](#technical-observations)

---

## Technical Context and Initialization

I recently conducted an analysis of a UEFI SMM driver that integrates an ARM TrustZone-based Firmware TPM (fTPM). The repository is private because it includes critical internal processes, handlers, and buffers, but the study allows for a detailed understanding of the communication between firmware, SMRAM, and TPM.

This driver acts as a bridge between the UEFI firmware and a *Trusted Application* in TrustZone. It does not implement the TPM's internal logic itself but ensures that all critical operations requested by the Operating System are processed securely and reliably.

The driver executes in System Management Mode (SMM), the CPU's most privileged level, operating invisibly to the operating system. Initialization occurs in two phases:

* **DXE Phase:** `InitializeSmmBase` salva handles globais essenciais, localiza protocolos necessários e verifica a existência de hardware fTPM via MMIO no endereço `0xE00D0000`.
* **SMM Phase:** `FtpmSmmInit` registra os handlers de *Software System Management Interrupts* (SW-SMI), instala a tabela ACPI TPM2 e configura os buffers de comando/resposta (CRB).

Driver persistence is maintained through `setjmp` and `longjmp`, allowing it to reside in memory even in the event of internal failures.

---

## Integrated Execution Flow

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

## Global Variables and Critical Structures

| Variável | Tipo | Description |
| :--- | :--- | :--- |
| `gImageHandle` | `EFI_HANDLE` | Handle da imagem do driver |
| `gSystemTable` | `EFI_SYSTEM_TABLE*` | Ponteiro para a tabela central do sistema |
| `gBS_local` | `EFI_BOOT_SERVICES*` | Boot services de escopo local |
| `gSmst` | `EFI_SMM_SYSTEM_TABLE2*` | Ponteiro para a Tabela SMM |
| `gSmmBase2` | `EFI_SMM_BASE2_PROTOCOL*` | Protocolo base da infraestrutura SMM |
| `gSmmVariable` | `EFI_SMM_VARIABLE_PROTOCOL*` | Protocolo para acesso e controle de variáveis SMM |
| `gInSmm` | `BOOLEAN` | Flag indicating whether the execution is inside SMRAM |
| `gAllocatedPool` | `VOID*` | Pre-allocated memory buffer dedicated to fTPM |
| `gFtpmDeviceFamily` | `UINT32` | Identificador numérico da família do dispositivo fTPM |
| `gFtpmData` | `FTPM_GLOBAL_DATA*` | Data shared for PP/MOR control |
| `gFtpmCrbSnapshot` | `TPM2_CRB_CONTROL_AREA` | Snapshot of CRB state |
| `gTpm2AcpiTable` | `TPM2_ACPI_TABLE` | TPM2 ACPI Table with a fixed size of 52 bytes |
| `gDriverStatus` | `EFI_STATUS` | Internal driver status and return code |
| `gJmpBuf` | `jmp_buf` | Stores CPU context for setjmp/longjmp |

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

## SW-SMI Handlers and Operating System Interaction

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

## Technical Observations

1. **Limitações de Endereçamento:** gFtpmData é truncado para 32 bits na ACPI.
2. **Controle Irrestrito:** execução em SMM permite controle total da memória física do sistema.
3. **Segurança e Mitigações:** buffers pequenos validados mitigam ausência de stack canaries.
4. **Sigilo de Informação:** código e processos internos permanecem privados, incluindo despacho de handlers e formatos de buffers críticos.
