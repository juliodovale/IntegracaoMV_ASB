Integração MV x ASB — Exportação de Dados Contábeis via EventBus

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

**8. Alteração dos dados após a sincronização**
-----------------------------------------------

O MV é a fonte dos dados contábeis. Portanto, alterações realizadas após uma exportação já processada pelo ASB deverão ser comunicadas.
O MV deverá publicar um evento, por exemplo: `AccountingDataChanged`
O evento deverá informar:
*   CNPJ;
    
*   período afetado;
    
*   identificadores dos registros alterados, quando disponíveis;
    
*   tipo da alteração (INSERT, UPDATE ou DELETE);
    
*   data/hora da alteração.
    

### Tratamento pelo ASB



    AccountingDataChanged
            ↓
    Identificar dados sincronizados afetados
            ↓
    Marcar sincronização como INCONSISTENTE
            ↓
    Verificar ECD/ECF relacionada
            ↓
    Se já gerada/transmitida → marcar como INCONSISTENTE
            ↓
    Permitir nova sincronização
Uma alteração mensal poderá afetar tanto uma sincronização mensal quanto uma anual que contenha aquele período.
A alteração não deverá ser aplicada silenciosamente sobre dados já sincronizados.

**9. Reprocessamento**
----------------------

Uma nova exportação poderá ser solicitada quando:
*   a exportação anterior falhar;
    
*   o arquivo estiver inválido ou indisponível;
    
*   os dados forem alterados após a sincronização;
    
*   a sincronização for marcada como inconsistente.
    
Cada nova exportação deverá gerar um novo `conversationId`.

**10. Contrato e mensageria**
-----------------------------

A comunicação utilizará o MassTransit/EventBus já adotado pela arquitetura.
Não deverá ser criado envelope customizado.
Os eventos oficiais são:
| **Evento** | **Origem** | **Destino** | **Finalidade** |
| --- | --- | --- | --- |
| `AccountingDataExportRequested` | ASB | MV | Solicitar exportação |
| `AccountingDataExportCompleted` | MV | ASB | Informar arquivo disponível |
| `AccountingDataExportFailed` | MV | ASB | Informar falha |
| `AccountingDataChanged` | MV | ASB | Informar alteração posterior dos dados |
O nome da classe e namespace dos eventos fazem parte do contrato e não devem ser alterados sem acordo entre os sistemas.

**11. Princípios**
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
