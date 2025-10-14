# 🧩 Order Processing System

> Sistema distribuído baseado em microsserviços com mensageria (Kafka),
> CI/CD e orquestração de containers. Desenvolvido em **Java 21 / Spring
> Boot 3**, com foco em escalabilidade, resiliência e boas práticas de
> arquitetura backend.

------------------------------------------------------------------------

## 🎯 Objetivo do Projeto

O **Order Processing System** simula o backend de um e-commerce moderno,
dividido em microsserviços independentes.\
Cada serviço possui seu próprio banco de dados e comunica-se com os
demais via **Apache Kafka**, garantindo **consistência eventual** e
**desacoplamento**.

O projeto demonstra:

-   Arquitetura orientada a eventos (Event-Driven Architecture)
-   Comunicação assíncrona com Kafka
-   Autenticação distribuída via JWT
-   Orquestração de containers com Docker Compose / Kubernetes
-   CI/CD automatizado com GitHub Actions
-   Persistência e logs distribuídos

------------------------------------------------------------------------

## 🏗️ Arquitetura do Sistema

### 🔹 Serviços Principais

  ---------------------------------------------------------------------------------
  Serviço                    Responsabilidade                Comunicação
  -------------------------- ------------------------------- ----------------------
  **user-service**           Gerencia usuários e             REST
                             autenticação JWT                

  **product-service**        Catálogo de produtos e          REST → Kafka
                             metadados                       

  **order-service**          Criação e gerenciamento de      REST → Kafka
                             pedidos                         

  **payment-service**        Processamento de pagamentos     Kafka

  **inventory-service**      Controle de estoque e reservas  Kafka

  **notification-service**   Envio de notificações e logs    Kafka

  **logging-service**        Centraliza logs técnicos        Kafka → PostgreSQL /
  *(opcional)*                                               Elasticsearch
  ---------------------------------------------------------------------------------

------------------------------------------------------------------------

## ⚙️ Fluxo Geral do Sistema

``` plaintext
Client → [user-service] → (JWT)
   ↓
[order-service] → Kafka (topic: orders)
   ↓
[payment-service] → Kafka (topic: payment-status)
   ↓
[inventory-service] → Kafka (topic: inventory-events)
   ↓
[notification-service] → Kafka (topic: notifications)
```

Cada serviço é autônomo, possui seu próprio banco e pode ser implantado
independentemente.

------------------------------------------------------------------------

## 📦 Descrição dos Serviços

### 🧍‍♂️ user-service

-   Registra e autentica usuários.
-   Gera tokens JWT usados nos outros serviços.
-   Banco: PostgreSQL (`users_db`)

### 🛒 product-service

-   Gerencia catálogo de produtos (nome, descrição, preço, categoria).
-   Publica eventos Kafka: `ProductCreated`, `ProductUpdated`,
    `ProductDeleted`.
-   Banco: PostgreSQL (`products_db`).

### 📦 inventory-service

-   Controla quantidades e reservas de produtos.
-   Consome eventos de `product-service` e `payment-service`.
-   Mantém consistência via eventos Kafka.
-   Banco: PostgreSQL (`inventory_db`) ou Redis.

### 🧾 order-service

-   Cria pedidos e armazena status localmente.
-   Publica eventos `OrderCreated` no Kafka.
-   Consome eventos de estoque e pagamento.
-   Banco: PostgreSQL (`orders_db`).

### 💳 payment-service

-   Consome `OrderCreated`, processa pagamento e publica
    `PaymentSuccess` ou `PaymentFailed`.
-   Banco: PostgreSQL (`payments_db`).

### 📬 notification-service

-   Escuta eventos `PaymentSuccess` e `InventoryUpdated`.
-   Envia notificações (ou registra logs).
-   Banco: PostgreSQL (`notifications_db`).

### 🧾 logging-service *(opcional)*

-   Consome eventos Kafka de logs e grava no PostgreSQL ou
    Elasticsearch.
-   Permite visualização centralizada dos logs no Kibana/Grafana.

------------------------------------------------------------------------

## 🔗 Fluxo de Negócio Completo

1.  **Usuário se autentica** no `user-service` e obtém JWT.\
2.  **Pedido é criado** via `order-service`.\
3.  `order-service` publica o evento `OrderCreated` no Kafka.\
4.  `payment-service` consome e processa o pagamento.\
5.  Em caso de sucesso, publica `PaymentSuccess`.\
6.  `inventory-service` consome e reduz o estoque.\
7.  `notification-service` envia confirmação ao usuário.\
8.  `logging-service` registra os logs e eventos de todo o processo.

------------------------------------------------------------------------

## 🧱 Integridade Referencial

-   **Interna (local):** Cada serviço usa chaves estrangeiras e
    constraints no próprio banco.\
-   **Entre serviços:** Mantida via **eventos Kafka** e **padrão
    Outbox**, garantindo consistência eventual.\
-   **Padrão Saga:** Coordena o fluxo entre pedidos, pagamentos e
    estoque.\
-   **Padrão Outbox:** Evita perda de eventos entre transações locais e
    publicações no Kafka.

------------------------------------------------------------------------

## 🐳 Orquestração (Docker Compose)

``` yaml
version: "3.9"
services:
  zookeeper:
    image: bitnami/zookeeper:latest
    environment:
      ALLOW_ANONYMOUS_LOGIN: yes

  kafka:
    image: bitnami/kafka:latest
    environment:
      KAFKA_CFG_ZOOKEEPER_CONNECT: zookeeper:2181
      ALLOW_PLAINTEXT_LISTENER: yes
    depends_on:
      - zookeeper

  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass

  order-service:
    build: ./order-service
    ports: ["8081:8080"]
    depends_on: [kafka, postgres]

  payment-service:
    build: ./payment-service
    depends_on: [kafka, postgres]

  inventory-service:
    build: ./inventory-service
    depends_on: [kafka, postgres]

  product-service:
    build: ./product-service
    depends_on: [kafka, postgres]

  notification-service:
    build: ./notification-service
    depends_on: [kafka]
```

------------------------------------------------------------------------

## ⚙️ CI/CD (GitHub Actions)

``` yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup JDK
        uses: actions/setup-java@v3
        with:
          java-version: '21'
      - name: Build & Test
        run: mvn clean verify

  docker:
    runs-on: ubuntu-latest
    needs: build-test
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker Images
        run: docker compose build
      - name: Push to Docker Hub
        run: |
          docker login -u ${{ secrets.DOCKER_USER }} -p ${{ secrets.DOCKER_PASS }}
          docker compose push
```

------------------------------------------------------------------------

## 🧠 Conceitos Aplicados

  -----------------------------------------------------------------------
  Conceito                         Descrição
  -------------------------------- --------------------------------------
  **DDD (Domain-Driven Design)**   Separação clara entre domínios:
                                   usuário, produto, pedido, estoque,
                                   pagamento.

  **Event-Driven Architecture**    Comunicação assíncrona via Kafka.

  **Sagas Pattern**                Coordenação distribuída de transações.

  **Outbox Pattern**               Publicação confiável de eventos após
                                   transações.

  **CI/CD Pipeline**               Build, teste e deploy automatizados
                                   via GitHub Actions.

  **Containerização**              Cada serviço tem seu Dockerfile e é
                                   orquestrado via Docker Compose.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 🧾 Logs e Observabilidade

  ------------------------------------------------------------------------
  Tipo de Log              Ferramenta              Finalidade
  ------------------------ ----------------------- -----------------------
  Logs de auditoria        PostgreSQL              Rastreabilidade e
                                                   histórico

  Logs técnicos            Elasticsearch /         Análise e depuração
                           OpenSearch              

  Métricas e alertas       Prometheus / Grafana    Monitoramento em tempo
                                                   real
  ------------------------------------------------------------------------

------------------------------------------------------------------------

## 🚀 Como Executar Localmente

``` bash
# Clonar o repositório
git clone https://github.com/usuario/order-processing-system.git
cd order-processing-system

# Subir containers
docker compose up --build

# Testar API (exemplo)
curl -X POST http://localhost:8081/orders -H "Authorization: Bearer <token>"
```

------------------------------------------------------------------------

## 📈 Roadmap Futuro

-   [ ] Adicionar Kubernetes (Helm Charts)
-   [ ] Implementar tracing com Zipkin
-   [ ] Adicionar testes de contrato (Spring Cloud Contract)
-   [ ] Implementar monitoramento completo (Grafana + Prometheus)
-   [ ] Criar dashboards de pedidos e estoque

------------------------------------------------------------------------

## 👨‍💻 Autor

**Mateus de Sousa Raimundo**\
Desenvolvedor Backend Java \| Spring Boot \| Kafka \| CI/CD \| Docker\
[GitHub](https://github.com/MateusdiSousa) \|
[LinkedIn](https://linkedin.com/in/mateussousara)
