# 📦 Pipeline de Ingestão de Pedidos com AWS Lambda, API Gateway, Amazon SQS FIFO e EventBridge

## 📖 Descrição

Neste laboratório foi desenvolvida a primeira etapa de uma arquitetura **Serverless orientada a eventos (Event-Driven Architecture)** utilizando serviços da AWS.

O fluxo implementado recebe pedidos através de uma API REST criada no Amazon API Gateway, realiza uma pré-validação utilizando uma função AWS Lambda, envia os pedidos para uma fila Amazon SQS FIFO garantindo o processamento em ordem e, posteriormente, uma segunda função Lambda realiza a validação completa antes de publicar um evento em um barramento customizado do Amazon EventBridge.

Essa arquitetura promove desacoplamento entre os componentes, maior escalabilidade e facilidade de integração com outros sistemas.

---

## 🎯 Objetivos

- Criar uma API REST utilizando Amazon API Gateway.
- Desenvolver funções AWS Lambda em Python.
- Configurar filas Amazon SQS FIFO e Dead Letter Queue (DLQ).
- Aplicar permissões utilizando IAM Roles.
- Implementar comunicação assíncrona entre serviços AWS.
- Publicar eventos personalizados utilizando Amazon EventBridge.
- Monitorar a execução através do Amazon CloudWatch.

---

## 🏗️ Arquitetura

> *Adicionar aqui a imagem da arquitetura construída durante o laboratório.*

<p align="center">
<img src="./imagens/arquitetura.png" width="900">
</p>

### Fluxo da aplicação

```text
Cliente

    │
    ▼

Amazon API Gateway

    │
    ▼

Lambda - Pré-Validação

    │
    ▼

Amazon SQS FIFO

    │
    ▼

Lambda - Validação

    │
    ▼

Amazon EventBridge
```

---

# 🛠️ Serviços AWS Utilizados

| Serviço | Finalidade |
|----------|------------|
| IAM | Gerenciamento de permissões das funções Lambda |
| AWS Lambda | Processamento e validação dos pedidos |
| Amazon API Gateway | Exposição da API REST |
| Amazon SQS FIFO | Enfileiramento e desacoplamento do processamento |
| Dead Letter Queue (DLQ) | Tratamento de falhas no processamento |
| Amazon EventBridge | Publicação de eventos da aplicação |
| Amazon CloudWatch | Monitoramento e logs das funções Lambda |

---

# 🚀 Desenvolvimento do Laboratório

## 1️⃣ Criação das IAM Roles

Foram criadas duas IAM Roles responsáveis pelas permissões das funções Lambda.

**Pré-Validação**

- AWSLambdaBasicExecutionRole

**Validação**

- AWSLambdaBasicExecutionRole

Posteriormente foram adicionadas permissões específicas para acesso ao Amazon SQS e ao Amazon EventBridge através de políticas inline.

> 📷 *Inserir print das Roles criadas.*

---

## 2️⃣ Configuração da Dead Letter Queue (DLQ)

Foi criada uma fila FIFO responsável por armazenar mensagens que não puderem ser processadas corretamente após três tentativas.

**Nome da fila**

```
pedidos-fifo-dlq.fifo
```

> 📷 *Inserir print da DLQ.*

---

## 3️⃣ Configuração da Fila Principal (FIFO)

A fila principal foi configurada utilizando o tipo **FIFO**, garantindo que os pedidos sejam processados exatamente na ordem em que forem recebidos.

Também foi associada à Dead Letter Queue para tratamento automático de falhas.

**Nome da fila**

```
pedidos-fifo-queue.fifo
```

> 📷 *Inserir print da fila principal.*

---

## 4️⃣ Atualização das Permissões IAM

Após a criação da fila SQS foram adicionadas políticas permitindo:

- Envio de mensagens para a fila (Lambda de Pré-Validação);
- Leitura e remoção de mensagens da fila (Lambda de Validação);
- Publicação de eventos no Amazon EventBridge.

> 📷 *Inserir print das políticas.*

---

## 5️⃣ Desenvolvimento da Lambda de Pré-Validação

Foi criada uma função Lambda responsável por receber os dados enviados pelo API Gateway.

Suas responsabilidades são:

- Receber o payload JSON.
- Validar campos obrigatórios.
- Enviar o pedido para a fila Amazon SQS FIFO.
- Retornar resposta HTTP ao cliente.

A URL da fila foi configurada utilizando variável de ambiente.

```python
SQS_QUEUE_URL = os.environ["SQS_QUEUE_URL"]
```

> 📷 *Inserir print da Lambda.*

---

## 6️⃣ Configuração do Amazon API Gateway

Foi criada uma API REST contendo o recurso:

```
/pedidos
```

com o método

```
POST
```

A integração foi realizada utilizando **Lambda Proxy Integration**, permitindo que toda requisição seja encaminhada diretamente para a Lambda de Pré-Validação.

Após a configuração foi realizado o deploy para o estágio:

```
dev
```

> 📷 *Inserir print do API Gateway.*

---

## 7️⃣ Testes da API

Após o deploy da API foi realizado um teste utilizando **curl**, enviando um pedido em formato JSON.

### Exemplo

```bash
curl -X POST <INVOKE_URL>/pedidos \
-H "Content-Type: application/json" \
-d '{
    "pedidoId":"001",
    "clienteId":"cliente01",
    "itens":[
        {
            "produto":"Caneta",
            "quantidade":10
        }
    ]
}'
```

### Resposta esperada

```json
{
    "message":"Pedido recebido e enfileirado",
    "sqsMessageId":"xxxxxxxxxxxxxxxx"
}
```

> 📷 *Inserir print do teste.*

---

## 8️⃣ Criação do Event Bus

Foi criado um barramento customizado utilizando Amazon EventBridge para publicação dos eventos gerados após a validação dos pedidos.

```
pedidos-event-bus
```

> 📷 *Inserir print do EventBridge.*

---

## 9️⃣ Desenvolvimento da Lambda de Validação

A segunda função Lambda é acionada automaticamente sempre que uma nova mensagem chega à fila SQS.

Suas responsabilidades são:

- Consumir mensagens da fila;
- Validar o conteúdo do pedido;
- Adicionar timestamp quando necessário;
- Publicar um evento no EventBridge.

O nome do Event Bus foi configurado através de variável de ambiente.

```python
EVENT_BUS_NAME = os.environ["EVENT_BUS_NAME"]
```

> 📷 *Inserir print da Lambda.*

---

## 🔟 Configuração do Trigger SQS

Foi configurado um Trigger entre a fila SQS FIFO e a Lambda de Validação.

Configuração utilizada:

- Batch Size = 1

Essa configuração garante que cada mensagem seja processada individualmente, preservando o comportamento FIFO.

> 📷 *Inserir print do Trigger.*

---

## 1️⃣1️⃣ Validação do Fluxo Completo

Após todos os recursos configurados foi realizado um novo envio de pedidos para validar toda a arquitetura.

Durante o teste foi possível verificar:

- API Gateway recebendo requisições;
- Lambda de Pré-Validação executando corretamente;
- Pedido sendo enviado para a fila SQS;
- Lambda de Validação consumindo a mensagem;
- Publicação do evento no Amazon EventBridge;
- Logs disponíveis no Amazon CloudWatch.

> 📷 *Inserir prints dos logs do CloudWatch.*

---

# 📊 Fluxo Final

```text
Cliente

        │
        ▼

API Gateway

        │
        ▼

Lambda Pré-Validação

        │
        ▼

Amazon SQS FIFO

        │
        ▼

Lambda Validação

        │
        ▼

Amazon EventBridge
```

---

# 📚 Aprendizados

Durante este laboratório foi possível compreender na prática conceitos importantes relacionados ao desenvolvimento de aplicações Serverless na AWS, incluindo:

- Arquiteturas orientadas a eventos;
- Processamento assíncrono;
- Comunicação desacoplada entre serviços;
- Utilização de filas FIFO;
- Dead Letter Queue (DLQ);
- IAM Roles e políticas de acesso;
- API Gateway integrado ao Lambda;
- Publicação de eventos no EventBridge;
- Monitoramento através do Amazon CloudWatch.

---

# ✅ Conclusão

Este laboratório permitiu implementar uma arquitetura Serverless baseada em eventos, utilizando serviços gerenciados da AWS para construir um fluxo de ingestão de pedidos escalável e desacoplado.

Ao final, foi possível integrar API Gateway, AWS Lambda, Amazon SQS FIFO e Amazon EventBridge em uma solução capaz de receber requisições, validar dados, realizar processamento assíncrono e publicar eventos para futuras integrações, seguindo boas práticas de arquitetura em Cloud Computing.

---
