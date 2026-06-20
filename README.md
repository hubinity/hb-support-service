# hb-support-service (Backend) - Ecossistema Hubinity - Planned

> Parte integrante do ecossistema distribuído Hubinity.
> ⚠️ **Status atual: Planned** — código de implementação ainda não foi escrito. Este README descreve o papel arquitetural pretendido conforme PRD seção 4 e roadmap em `docs/phases/`.

---

## 💻 Visão Geral

- **O que faz:** Gestão de chamados técnicos da HiBit. Cadastra técnicos, clientes e o catálogo de serviços (com preço base); abre tickets, distribui automaticamente por carga + especialidade, acompanha o ciclo de execução e finaliza o atendimento. Ao fechar, publica `ServiceRevenueGenerated` para o caixa lançar a receita do serviço.
- **Problema que resolve:** acaba com a distribuição manual de chamados (que gera ociosidade de técnicos e SLA inconsistente) e com a digitação dupla da receita de serviços no livro-caixa.
- **Posicionamento no Ecossistema:** serviço autônomo de operação interna HiBit. Não depende de catálogo nem de pedidos do totem — apenas alimenta o caixa com receita de serviços.

## 🏗️ Papel na Arquitetura

- **Tipo de Componente:** Microsserviço Spring Boot, database-per-service (Postgres dedicado no Supabase), publicador via Transactional Outbox.
- **Responsabilidades Principais (planejadas):**
  - CRUD de `Technician`, `Customer`, `Service`.
  - Ciclo de vida do `Ticket` (`open` → `assign` → `start` → `close`, com `reassign` a qualquer momento).
  - Auditoria via `TicketEvent` (ASSIGNED/STARTED/PAUSED/RESUMED/CLOSED/REASSIGNED).
  - Distribuidor automático: round-robin ponderado pela `currentLoad` filtrando técnicos por `specialty ⊇ services_required`.
  - Publicação de `ServiceRevenueGenerated` via outbox quando ticket é fechado com ao menos 1 item.
- **Limites e Fronteiras (Boundaries):** não toca caixa diretamente (publica evento), não emite cobrança ao cliente final (futuro), não orquestra estoque de peças (também futuro).

## 🔗 Dependências e Comunicação (Planejadas)

### Serviços Internos da Hubinity

- **`hb-cashier-service`** — consumidor (via broker) de `support.events.ServiceRevenueGenerated`.
- **`platform-iam` (Keycloak)** — valida JWT da realm `hibit`; roles `admin`, `tecnico`, `atendente`.
- **`platform-shared-contracts`** — depende de `contracts-support` e `contracts-events`.

### Infraestrutura e Serviços Externos

- **Supabase** — projeto Postgres dedicado `hb-support`.
- **CloudAMQP** — RabbitMQ compartilhado (publicação em `support.events`).
- **Railway** — host de deploy.

## 🛠️ Tecnologias e Ferramentas (Stack Prevista)

| Camada | Tecnologia | Versão |
| :--- | :--- | :--- |
| Linguagem | Java | 21 (LTS) |
| Build | Maven | 3.9+ |
| Framework | Spring Boot | 4.1 |
| Módulos Spring | Web, Data JPA, Flyway, AMQP, Security, Resource Server | — |
| Mapper | MapStruct | 1.6 |
| Banco | PostgreSQL (Supabase) | 15+ |
| Broker | RabbitMQ (CloudAMQP) | 3.x |
| Testes | JUnit 5 + Testcontainers (Postgres + RabbitMQ) | última estável |
| Container | Docker (multi-stage) | — |

## 📐 Padrões de Projeto e Arquitetura do Código (Previstos)

- **Estilo Arquitetural:** Microsserviço autônomo Spring Boot, mesma base do catalog/cashier.
- **Padrões Relevantes:**
  - **Transactional Outbox** para `ServiceRevenueGenerated` (sem perder evento se broker estiver fora).
  - **Domain Events** — `TicketEvent` é a fonte de auditoria temporal do ticket (event log mínimo dentro do agregado).
  - **Strategy** para distribuição (atual: round-robin ponderado; pluggable para futuras estratégias por SLA ou geografia).
  - **Database-per-service** sem FK cross-service.

## 🗺️ Roadmap & Posição no Board

- **Fase do PRD:** Fase 4 — Chamados (PRD seção 9).
- **Tasks no board:**
  - `4.1` — Bootstrap.
  - `4.2` — Flyway: `customer`, `technician`, `service`, `ticket`, `ticket_item`, `ticket_event`.
  - `4.3` — CRUDs entidades de cadastro.
  - `4.4` — Distribuidor automático (round-robin ponderado) + testes unitários.
  - `4.5` — Endpoints de ciclo de vida (open/assign/start/close/reassign).
  - `4.6` — Publicação `ServiceRevenueGenerated` via outbox.
  - `4.7` — Consumer correspondente no caixa.
- **Dependências bloqueadoras:** `hb-cashier-service` em staging (Fase 2) — sem ele, o evento de receita não tem destino útil. Skeleton do consumer (Fase 2.5) precisa existir antes da 4.7.

## ⚙️ Variáveis de Ambiente (Previstas)

```bash
# App
SPRING_PROFILES_ACTIVE=staging
SERVER_PORT=8082

# Banco — Supabase (projeto hb-support)
SPRING_DATASOURCE_URL=jdbc:postgresql://<host>.pooler.supabase.com:6543/postgres?sslmode=require
SPRING_DATASOURCE_USERNAME=
SPRING_DATASOURCE_PASSWORD=

# Broker
SPRING_RABBITMQ_HOST=
SPRING_RABBITMQ_PORT=5671
SPRING_RABBITMQ_SSL_ENABLED=true
SPRING_RABBITMQ_USERNAME=
SPRING_RABBITMQ_PASSWORD=
SPRING_RABBITMQ_VIRTUAL_HOST=

# IAM — Keycloak (realm hibit)
KEYCLOAK_ISSUER_URI=https://iam.hubinity.app/realms/hibit
```

## 🚀 Como Será Executado (Quando Implementado)

### Pré-requisitos

- JDK 21, Maven 3.9+
- Stack local de `platform-infra` rodando

### Execução (Será disponível após bootstrap da Fase 4)

```bash
SPRING_PROFILES_ACTIVE=local mvn spring-boot:run

mvn -B verify

docker build -t ghcr.io/hubinity/hb-support-service:dev .
```
