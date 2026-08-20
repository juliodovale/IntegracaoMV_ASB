Integração MV x ASB — Exportação de Dados Contábeis via EventBus
==

**1. Objetivo**
---------------

Disponibilizar uma integração assíncrona para que o ASB solicite ao MV dados contábeis de um CNPJ e período, destinados à geração da ECD e ECF.
O MV realiza a extração e disponibiliza o resultado em um Storage. O ASB é notificado pelo EventBus, realiza o download e processa os dados.
O arquivo de exportação não trafega pelo EventBus.

**2. Arquitetura**
------------------


                             ASB
                              │
                              │ AccountingDataExportRequested
                              ▼
                         ┌─────────┐
                         │ EventBus│
                         └────┬────┘
                              │
                              ▼
                             MV
                        Valida / Processa
                        Gera arquivo ZIP
                              │
                              │ Upload
                              ▼
                         ┌─────────┐
                         │ Storage │
                         └────┬────┘
                              │
                              │ Arquivo disponível
                              ▼
                         ┌─────────┐
                         │ EventBus│
                         └────┬────┘
                              │
                              │ AccountingDataExportCompleted
                              ▼
                             ASB
                        Download / Validação
                        Descompactação
                        Processamento
                              │
                              ▼
                        Dados disponíveis

### Responsabilidades

| **Componente** | **Responsabilidade** |
| --- | --- |
| ASB | Solicitar e processar os dados |
| MV | Extrair e gerar o arquivo |
| EventBus | Transportar eventos e metadados |
| Storage | Armazenar o arquivo |

**3. Solicitação da exportação**
--------------------------------

O ASB publica o evento:
`AccountingDataExportRequested`
A solicitação deverá informar:
*   CNPJ;
    
*   período;
   
    
O período poderá ser:
*   **Anual**
    JSON
    
        {
          "type": "YEAR",
          "year": 2026,
          "month": null
        }
    
*   **Mensal**
    JSON
    
        {
          "type": "MONTH",
          "year": 2026,
          "month": 7
        }
    
O envelope da mensagem seguirá o padrão do MassTransit. O contrato funcional será definido pelo conteúdo de `message`.

**4. Identificação e rastreabilidade**
--------------------------------------

Cada solicitação deverá possuir um `conversationId` único, gerado pelo ASB.
O MV deverá preservar esse identificador durante todo o processamento e retorná-lo nos eventos:
*   `AccountingDataExportCompleted`
    
*   `AccountingDataExportFailed`
    
O `conversationId` será utilizado pelo ASB para identificar a operação e garantir idempotência em caso de reentrega da mesma mensagem.
Uma nova solicitação para o mesmo CNPJ e período deverá possuir um novo `conversationId`.

**5. Processamento pelo MV**
----------------------------

Ao receber `AccountingDataExportRequested`, o MV deverá:

    Receber solicitação
           ↓
    Validar CNPJ e período
           ↓
    Registrar solicitação
           ↓
    Processar dados
           ↓
    Gerar arquivo
           ↓
    Compactar ZIP
           ↓
    Enviar para Storage
           ↓
    Publicar resultado
O processamento será assíncrono.

### Sucesso

Publicar: `AccountingDataExportCompleted`
O evento deverá informar, no mínimo:
JSON

    {
      "sourceIdentifier": "MV",
      "document": {
        "type": "CNPJ",
        "value": "55233019000100"
      },
      "period": {
        "type": "YEAR",
        "year": 2026,
        "month": null
      },
      "status": "COMPLETED",
      "conversationId": "ABC-123",
      "file": {
        "name": "fiscal-export-55233019000100-2026.zip",
        "format": "JSON",
        "compression": "ZIP",
        "sizeBytes": 460336005,
        "storageProvider": "S3",
        "bucket": "mv-asb-fiscal-export",
        "key": "2026/08/18/55233019000100/ABC-123/fiscal-export.zip",
        "checksum": {
          "algorithm": "SHA-256",
          "value": "..."
        }
      },
      "processedAt": "2026-08-18T14:15:32Z"
    }
O ASB deverá validar CNPJ, período, `conversationId`, tamanho e checksum antes de processar o arquivo.

### Falha

Em caso de erro, o MV deverá publicar: `AccountingDataExportFailed`
Contendo:
JSON

    {
      "sourceIdentifier": "MV",
      "document": {
        "type": "CNPJ",
        "value": "55233019000100"
      },
      "period": {
        "type": "YEAR",
        "year": 2026,
        "month": null
      },
      "status": "FAILED",
      "conversationId": "ABC-123",
      "error": {
        "code": "EXPORT_PROCESSING_ERROR",
        "message": "Não foi possível gerar o arquivo."
      },
      "occurredAt": "2026-08-18T14:15:32Z"
    }

**6. Storage**
--------------

O arquivo será armazenado em Storage, preferencialmente S3 quando a infraestrutura estiver na AWS.

### Estrutura sugerida


    mv-asb-fiscal-export/
    └── {ano}/
        └── {mes}/
            └── {CNPJ}/
                └── {conversationId}/
                    └── fiscal-export.zip
O `conversationId` no caminho evita sobrescrita e permite rastrear cada exportação.

**7. Processamento pelo ASB**
-----------------------------

Após receber `AccountingDataExportCompleted`:

    Receber evento
         ↓
    Validar operação
         ↓
    Verificar idempotência
         ↓
    Localizar arquivo
         ↓
    Download
         ↓
    Validar tamanho/checksum
         ↓
    Descompactar
         ↓
    Validar dados
         ↓
    Persistir
         ↓
    Disponibilizar para ECD/ECF
O processamento deverá ser idempotente para suportar reentrega de mensagens e retries.

**8. Sincronização dos períodos contábeis
------------------------------------------
O MV deverá comunicar ao ASB, por meio do EventBus, alterações no estado dos períodos contábeis mensais.

### Eventos

* `AccountingPeriodCreated` — período criado;
* `AccountingPeriodClosed` — período fechado;
* `AccountingPeriodReopened` — período reaberto;
* `AccountingPeriodDeleted` — período excluído.

### Dados do evento

```json
{
  "cnpj": "55233019000100",
  "periodo": "2026-07",
  "changedAt": "2026-08-20T14:35:00Z"
}
```

### Tratamento pelo ASB

Ao receber o evento, o ASB deverá identificar as sincronizações que abrangem o período informado e, quando aplicável, atualizar seu estado e os processos de ECD/ECF relacionados.

```text
AccountingPeriodReopened
        ↓
Identificar sincronizações que abrangem o período
        ↓
Marcar como INCONSISTENTE
        ↓
Verificar ECD/ECF relacionada
        ↓
Se já gerada/transmitida → marcar como INCONSISTENTE
```

O MV é responsável apenas por comunicar a alteração do estado do período. Cabe ao ASB identificar as sincronizações afetadas.

Uma alteração em um período poderá afetar tanto uma sincronização mensal quanto uma sincronização anual que contenha aquele período.


**9. Reprocessamento**
----------------------

Uma nova exportação poderá ser solicitada quando:
*   a exportação anterior falhar;
    
*   o arquivo estiver inválido ou indisponível;
    
*   os dados forem alterados após a sincronização;
    
*   a sincronização for marcada como inconsistente.
    
Cada nova exportação deverá gerar um novo `conversationId`.

**10. Contrato e mensageria**
----------------------------

A comunicação utilizará o MassTransit/EventBus já adotado pela arquitetura.

Não deverá ser criado envelope customizado.

Os eventos oficiais são:

| **Evento**                      | **Origem** | **Destino** | **Finalidade**                                          |
| ------------------------------- | ---------- | ----------- | ------------------------------------------------------- |
| `AccountingDataExportRequested` | ASB        | MV          | Solicitar exportação                                    |
| `AccountingDataExportAvailable` | MV         | ASB         | Informar que a exportação está disponível para consulta |
| `AccountingDataExportFailed`    | MV         | ASB         | Informar falha na exportação                            |
| `AccountingPeriodCreated`       | MV         | ASB         | Informar criação do período contábil                    |
| `AccountingPeriodClosed`        | MV         | ASB         | Informar fechamento do período contábil                 |
| `AccountingPeriodReopened`      | MV         | ASB         | Informar reabertura do período contábil                 |
| `AccountingPeriodDeleted`       | MV         | ASB         | Informar exclusão do período contábil                   |

O nome das classes e seus namespaces fazem parte do contrato e não devem ser alterados sem acordo entre os sistemas.

**11. Garantia de entrega dos eventos
--------------------------------------

O MV deverá utilizar o Outbox Pattern para garantir a consistência entre as alterações realizadas no banco de dados e a publicação dos eventos no EventBus.

A alteração dos dados e o registro do evento na Outbox deverão ocorrer na mesma transação. A publicação no EventBus será realizada posteriormente, permitindo o reprocessamento de mensagens em caso de falha na comunicação.

                         MV
                          │
                          │ Atualiza dados
                          ▼
                    ┌───────────┐
                    │  Outbox   │
                    └─────┬─────┘
                          │
                          │ Registra evento
                          ▼
                       COMMIT
                          │
                          ▼
                    ┌───────────┐
                    │  Outbox   │
                    └─────┬─────┘
                          │
                          │ Publicação
                          ▼
                    ┌───────────┐
                    │ EventBus  │
                    └─────┬─────┘
                          │
                          │ Evento
                          ▼
                         ASB

**12. Princípios**
------------------

1.  **Assíncrono:** o ASB não aguarda o processamento do MV.
    
2.  **EventBus:** transporta eventos e metadados, nunca o arquivo.
    
3.  **Storage:** responsável pelo arquivo de exportação.
    
4.  **Rastreabilidade:** `conversationId` acompanha cada solicitação.
    
5.  **Idempotência:** reentregas não podem gerar processamento duplicado.
    
6.  **Integridade:** o ASB valida o arquivo antes da importação.
    
7.  **Consistência:** alterações posteriores no MV devem ser propagadas ao ASB.
    
8.  **Reprocessamento:** dados inconsistentes devem poder ser sincronizados novamente.
    
9.  **Observabilidade:** CNPJ, período e `conversationId` devem permitir rastrear a operação ponta a ponta.
