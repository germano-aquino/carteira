# Plano de estudo: Spring Boot (projeto "Carteira Digital")

Criado em 2026-11-05, revisado em 2026-11-09. Este é o único arquivo do plano: ele juntou `plano-spring-boot.md` e a versão estendida.

**Ritmo base:** 6 h/semana, em 3 blocos de 2 h (estudo, construção e testes/CI). Com 10 h, faça também os itens marcados com **(+)** e adiante semanas: o plano cai para cerca de 9 semanas.

---

## 1. Por que este projeto

Um domínio financeiro conversa direto com PagBank (#1), Cresol (#5, riscos de crédito) e com os clientes financeiros da CI&T (#2, #10). Ele obriga a cobrir os gaps que mais aparecem nas vagas Java, nesta ordem:

1. Spring Boot + Java moderno (17/21+)
2. JPA / Spring Data e performance de SQL
3. Microsserviços
4. Mensageria com consumidor idempotente, retry, DLQ e Outbox
5. JUnit 5 + Mockito + Testcontainers
6. Docker e CI
7. Resiliência, observabilidade e segurança

**O que você já sabe e não precisa reestudar:** PostgreSQL, migrations, Docker Compose, GitHub Actions, testes de integração (os 290 testes em Jest), TDD e Java da SAP. O foco fica no ecossistema Spring, mas o Java core entra numa trilha curta (seção 3), porque entrevista técnica cobra isso diretamente.

---

## 2. Decisões técnicas (tomar na semana 1)

| Decisão | Escolha | Por quê |
|---|---|---|
| Versão do Spring Boot | **Spring Boot 4.x** (Spring Framework 7) | Em nov/2026 o Initializr já sugere a 4.x por padrão e a linha 3.x está saindo do suporte gratuito. Os conceitos são os mesmos. No CV: "Spring Boot 3/4". |
| Java | **21 LTS** (ou 25 LTS) | As vagas pedem 17+, e a 21 cobre a 17. Virtual threads e pattern matching já estão disponíveis. |
| Build | **Maven**, com `./mvnw` | É o mais comum nas vagas. |
| Repositório | **Monorepo multi-módulo Maven** | Um CI só, um `docker compose up` só, e o recrutador abre um link só. |
| Dinheiro | **`BigDecimal`** (coluna `NUMERIC(19,2)`), nunca `double` | `0.1 + 0.2 != 0.3`. Isso é pergunta clássica de entrevista em fintech. |
| Modelo de saldo | **Ledger (lançamentos)** | O extrato sai dos lançamentos e o saldo é derivado ou mantido como cache com lock. Uma coluna `saldo` que muda sem histórico não é auditável. |

**Atenção às diferenças da versão 4 em tutoriais antigos:** muitos artigos (Baeldung inclusive) foram escritos para a versão 3. Na 4 mudam nomes de starters (ex.: `spring-boot-starter-webmvc`), o Jackson passa para a versão 3 (pacote `tools.jackson`) e o Testcontainers vai para a 2.x. Quando um exemplo não compilar, confira primeiro a versão.

**Estrutura do repositório:**

```
carteira/
├── pom.xml                  (pai, packaging pom)
├── wallet-api/
├── notification-service/    (semana 6)
├── antifraude-mock/         (semana 9)
├── docker-compose.yml
└── .github/workflows/ci.yml
```

---

## 3. Arquitetura final

| Serviço / Infra | Responsabilidade | Stack |
|---|---|---|
| **wallet-api** | Contas, depósitos, transferências, lançamentos e extrato | Spring Boot, Java 21, Spring Data JPA, PostgreSQL, Flyway, Spring Security |
| **notification-service** | Consome `TransferenciaConcluida` e registra/envia a notificação, sem duplicar | Spring Boot, Spring AMQP |
| **antifraude-mock** | Serviço lento e instável de propósito, para exercitar circuit breaker | Spring Boot |
| **PostgreSQL** | Contas, lançamentos, outbox e eventos processados | PostgreSQL 16 + Flyway |
| **RabbitMQ** | Exchange de eventos, filas de retry e DLQ | RabbitMQ (AMQP 0-9-1) |
| **Infra / CI** | Sobe tudo localmente e roda os testes em todo PR | Docker Compose, GitHub Actions |

### Fluxo de uma transferência

1. O cliente chama `POST /transferencias` (via REST ou pelo front) com o header `Idempotency-Key`.
2. A `wallet-api` valida o JWT e o payload.
3. Se a chave já foi processada, a API devolve a resposta original e para por aqui.
4. A `wallet-api` consulta o `antifraude-mock` **fora da transação**, protegida por circuit breaker (Resilience4j).
5. Em uma única transação: lock das contas, débito e crédito como lançamentos, gravação do evento na tabela **outbox** e registro da chave de idempotência.
6. Um publicador lê a outbox e envia `TransferenciaConcluida` ao RabbitMQ (*at-least-once*).
7. O `notification-service` consome o evento, confere a tabela de eventos processados (consumidor idempotente) e grava a notificação. Falhas repetidas vão para a DLQ.

---

## 4. Trilha contínua: Java core para entrevista

Cerca de 30 min por semana, dentro do bloco de estudo. É revisão, não código novo do projeto. Use as próprias classes do projeto como exemplo sempre que puder.

| Sem. | Tema |
|---|---|
| 1 | `equals`/`hashCode`/`toString`, imutabilidade, records |
| 2 | Collections: `List`/`Set`/`Map`, `HashMap` por dentro, `ArrayList` vs `LinkedList` |
| 3 | Streams e Optional (e quando não usar) |
| 4 | Concorrência: `synchronized`, `volatile`, `java.util.concurrent`, race condition |
| 5 | Exceções checked vs unchecked e por que o Spring faz rollback só em unchecked |
| 6 | Generics, interfaces funcionais, lambdas |
| 7 | SOLID com exemplos do próprio projeto |
| 9 | Virtual threads (Java 21) e quando elas ajudam em I/O |
| 10 | JVM: heap vs stack, garbage collector, `OutOfMemoryError` |
| 11 | Design patterns que o Spring usa: proxy, template method, strategy |
| 12 | Arquitetura em camadas vs hexagonal, e onde o projeto se encaixa |

---

## 5. Semanas

| Sem. | Início | Tema | Gap que fecha |
|---|---|---|---|
| 1 | 09/11 | Setup, decisões e fundamentos do Spring | Spring Boot, CI |
| 2 | 16/11 | Spring Data JPA, Flyway e modelo de dinheiro | JPA |
| 3 | 23/11 | API REST bem-feita | Spring MVC |
| 4 | 30/11 | Transações e concorrência | regra de negócio financeira |
| 5 | 07/12 | Idempotência da API + testes a fundo | idempotência, JUnit + Mockito |
| 6 | 14/12 | 2º serviço + RabbitMQ (versão ingênua) | microsserviços, mensageria |
| — | 21/12 | **Folga (festas)** | — |
| 7 | 28/12 | Mensageria confiável: Outbox, consumidor idempotente, retry e DLQ | mensageria (o que as vagas cobram) |
| 8 | 04/01 | **Checkpoint**: CV, vagas e decisão Kafka vs Angular | — |
| 9 | 11/01 | Chamada síncrona + Resilience4j | circuit breaker |
| 10 | 18/01 | Docker das apps + observabilidade | Docker, observabilidade |
| 11 | 25/01 | Spring Security + JWT | segurança |
| 12 | 01/02 | Vitrine: README, decisões e Kubernetes básico | apresentação, Kubernetes (extra) |
| 13–14 | 08/02 | Kafka **ou** Angular (decidido na semana 8) | o que as vagas pedirem |

> **Testes desde a semana 1.** Você já trabalha com TDD (use a skill `/tdd`). Todo "Pronto quando" é provado por um teste automatizado no CI, salvo quando o critério for visual (logs, Swagger, Zipkin). A semana 5 aprofunda os testes, mas não é onde eles começam.

---

### Semana 1: Setup, decisões e fundamentos

- **Estudar (2 h):** IoC e injeção de dependência, ciclo de vida de beans, `@Configuration`, profiles, `application.yml`, como funciona o auto-configuration (rode com `--debug` para ver o relatório).
- **Construir (3 h):** `git init`; monorepo Maven com pom pai e módulo `wallet-api`, gerado no Spring Initializr com Spring Boot 4.x (Web, Data JPA, PostgreSQL, Validation, Actuator, Flyway, Testcontainers); Postgres no Docker Compose; endpoint `GET /ping`.
- **Testes/CI (1 h):** workflow do GitHub Actions rodando `./mvnw verify` em todo PR; primeiro teste (`@SpringBootTest` sobe o contexto e `/ping` responde).
- **Pronto quando:** o repo público tem CI verde, `/actuator/health` responde `UP` e existe pelo menos um teste rodando no CI.
- **(+)** Ler a seção "Core Features" da documentação do Spring Boot.

**Leituras:**
- Spring Boot Reference: Core Features — https://docs.spring.io/spring-boot/reference/features/index.html
- Baeldung: "Intro to Inversion of Control and Dependency Injection with Spring"
- Baeldung: "Create a Custom Auto-Configuration with Spring Boot"

---

### Semana 2: Spring Data JPA, Flyway e modelo de dinheiro

- **Estudar:** entidades, `@OneToMany`/`@ManyToOne`, lazy vs eager, problema N+1, `JpaRepository`, query methods, `@Query`. **Modelagem de dinheiro:** `BigDecimal` (`compareTo` vs `equals`, `setScale`, `RoundingMode`) e por que usar ledger (lançamentos) em vez de só uma coluna de saldo.
- **Construir:** entidades `Cliente`, `Conta` e `Lancamento` (tipo DÉBITO/CRÉDITO, valor, data, referência da operação); migrations Flyway (o equivalente ao node-pg-migrate que você já usa), começando por `V1__create_tables.sql`; CRUD de contas; saldo calculado a partir dos lançamentos.
- **Testes:** `@DataJpaTest` com Postgres real via Testcontainers (`@ServiceConnection`), sem H2: H2 esconde diferenças de SQL e de lock.
- **Pronto quando:** você mostra no log do Hibernate uma query N+1 e a correção com `JOIN FETCH` ou `@EntityGraph`, e um teste prova que o saldo é igual à soma dos lançamentos.
- **(+)** Projeções com records e paginação com `Pageable`.

**Leituras:**
- Baeldung: "Database Migrations with Flyway"
- Baeldung: "N+1 Problem in Hibernate and Spring Data JPA"
- Testcontainers guides (Java + Spring Boot) — https://testcontainers.com/guides/

---

### Semana 3: API REST bem-feita

- **Estudar:** DTOs com records, Bean Validation (`@NotNull`, `@Positive`, `@Valid`), `@RestControllerAdvice`, `ProblemDetail` (RFC 9457), códigos HTTP corretos (400 vs 404 vs 409 vs 422).
- **Construir:** endpoints de depósito e de extrato paginado (lido dos lançamentos); erros padronizados; documentação com springdoc-openapi (Swagger UI). Confira se a versão do springdoc é compatível com o Spring Boot 4.
- **Testes:** `@WebMvcTest` + `MockMvc` (ou `MockMvcTester`) para os controllers, com o service mockado.
- **Pronto quando:** um payload inválido devolve 400 com `ProblemDetail` legível (provado por teste), e o Swagger mostra todos os endpoints.
- **(+)** MapStruct para mapear entidade ↔ DTO.

**Leituras:**
- RFC 9457 — https://www.rfc-editor.org/rfc/rfc9457
- Spring Framework Reference: "Error Responses" (seção de Web MVC)
- Baeldung: "Documenting a Spring REST API Using OpenAPI 3.0"

---

### Semana 4: Transações e concorrência (o coração do projeto)

> É o assunto que mais rende conversa em entrevista de fintech.

- **Estudar:** `@Transactional` (propagação, rollback só em unchecked por padrão, por que não funciona em chamada interna: é um proxy), níveis de isolamento, lock otimista com `@Version` vs lock pessimista (`PESSIMISTIC_WRITE` / `SELECT ... FOR UPDATE`), deadlock e por que travar as contas sempre na mesma ordem (por id).
- **Construir:** `POST /transferencias` gravando débito e crédito como lançamentos na mesma transação; regra de saldo insuficiente; lock das duas contas em ordem determinística.
- **Testes:** teste de integração com Testcontainers que dispara N transferências concorrentes (`ExecutorService` + `CountDownLatch`) na mesma conta.
- **Pronto quando:** o teste de concorrência passa no CI, o saldo final está correto e nenhuma conta fica negativa.
- **(+)** Implementar as duas versões (otimista com retry e pessimista) e comparar no README.

**Leituras:**
- Baeldung: "Transactions with Spring and JPA"
- Baeldung: "Optimistic Locking in JPA" e "Pessimistic Locking in JPA"
- Spring Framework Reference: "Transaction Management" — https://docs.spring.io/spring-framework/reference/data-access/transaction.html

---

### Semana 5: Idempotência da API + testes a fundo

A `Idempotency-Key` saiu da semana 4 porque ela sozinha tem detalhes suficientes para uma semana.

- **Estudar:**
  - **Idempotência:** onde guardar a chave (tabela com `UNIQUE`); o que guardar (status e corpo da resposta); o que fazer com duas requisições simultâneas com a mesma chave (a constraint unique decide quem ganha, o perdedor espera ou recebe 409); mesma chave com payload diferente (422); expiração.
  - **Testes:** Mockito a fundo (`@Mock`, `@InjectMocks`, `verify`, `ArgumentCaptor`, quando *não* mockar), test slices, JaCoCo.
- **Construir:** suporte ao header `Idempotency-Key` em `POST /transferencias`, com o registro da chave dentro da mesma transação da transferência; testes unitários do service de transferência; JaCoCo publicando a cobertura no CI.
- **Pronto quando:** um teste envia a mesma requisição duas vezes (inclusive em paralelo) e só uma transferência acontece, a segunda recebe a resposta original, e o CI publica a cobertura.

**Leituras:**
- Spring Boot Reference: "Testing" — https://docs.spring.io/spring-boot/reference/testing/index.html
- Stripe API docs: "Idempotent requests" (referência de mercado para o desenho da chave)

---

### Semana 6: 2º serviço + RabbitMQ (versão ingênua, de propósito)

- **Estudar:** por que microsserviços (e quando **não** usar), exchanges (direct, fanout, topic), filas, routing keys, Spring AMQP, serialização JSON de eventos. **O problema do dual write:** gravar no banco e publicar no broker não são atômicos.
- **Construir:** módulo `notification-service`; a `wallet-api` publica `TransferenciaConcluida` com `@TransactionalEventListener(phase = AFTER_COMMIT)` e o outro serviço consome e grava.
- **Experimento obrigatório:** derrube o RabbitMQ logo depois de uma transferência e veja o evento se perder. Anote o que aconteceu: é o "antes" da história da semana 7.
- **Pronto quando:** `docker compose up` sobe Postgres, RabbitMQ e os dois serviços, uma transferência aparece no `notification-service`, e o experimento de perda está documentado.

**Leituras:**
- RabbitMQ Tutorials (versão Spring AMQP) — https://www.rabbitmq.com/tutorials
- Spring Guide: "Messaging with RabbitMQ" — https://spring.io/guides/gs/messaging-rabbitmq
- Spring AMQP Reference — https://docs.spring.io/spring-amqp/reference/

---

### Semana de folga (21/12)

Se sobrar tempo, use para tirar atrasos. Se não, descanse: o plano já conta com ela.

---

### Semana 7: Mensageria confiável

- **Estudar:** entrega at-least-once, por que o consumidor precisa ser idempotente, retry com backoff exponencial, dead-letter queue, padrão **Transactional Outbox**.
- **Construir:**
  - **Outbox (obrigatório):** a `wallet-api` grava o evento na tabela `outbox` dentro da transação da transferência; um publicador agendado (`@Scheduled`, `SELECT ... FOR UPDATE SKIP LOCKED`) envia ao RabbitMQ e marca como enviado.
  - **Consumidor idempotente:** tabela de eventos processados no `notification-service` (o id do evento é a chave única).
  - Retry com backoff e DLQ para mensagens que falham sempre.
- **Testes:** Testcontainers com RabbitMQ.
- **Pronto quando:**
  - repetir o experimento da semana 6 (derrubar o RabbitMQ) não perde o evento: ele é enviado quando o broker volta;
  - um teste envia o mesmo evento duas vezes e confere que há só um registro;
  - uma mensagem "venenosa" termina na DLQ.
- **(+)** Comparar com Debezium/CDC no README (só a explicação, não a implementação).

**Leituras:**
- microservices.io: Transactional Outbox — https://microservices.io/patterns/data/transactional-outbox.html
- microservices.io: Idempotent Consumer — https://microservices.io/patterns/communication-style/idempotent-consumer.html
- Baeldung: "Error Handling with Spring AMQP"
- RabbitMQ docs: "Dead Letter Exchanges" — https://www.rabbitmq.com/docs/dlx

---

### Semana 8: Checkpoint (sem código novo)

- Adicionar o projeto ao CV pelo modo `add`, com o que foi feito de verdade até aqui (e LinkedIn).
- Reavaliar as vagas **ainda abertas**: PagBank (#1), Cresol (#5), e a #8, #9 e #10 pelo Scrum e pelos bugs em produção.
- Rodar um scan novo: as vagas de hoje provavelmente terão fechado. O plano mira o padrão das vagas, não estas específicas.
- **Decidir as semanas 13–14:** contar quantas vagas do scan pedem **Kafka** e quantas pedem **Angular**, e escolher a que aparecer mais.
- Montar a narrativa de entrevista com as três histórias até aqui: concorrência (semana 4), idempotência da API (semana 5) e outbox "antes e depois" (semanas 6–7).
- Decidir se vale começar a aplicar já, em paralelo com as semanas seguintes.
- **Pronto quando:** README atualizado com o progresso, CV pronto para enviar e decisão das semanas 13–14 anotada neste arquivo.

---

### Semana 9: Chamada síncrona + Resilience4j

- **Estudar:** `RestClient` (ou interfaces HTTP declarativas com `@HttpExchange`), timeouts de conexão e de leitura, circuit breaker, retry, bulkhead, fallback.
- **Construir:** módulo `antifraude-mock`, que às vezes demora ou falha (configurável); a `wallet-api` consulta antes de transferir, com circuit breaker e fallback.
- **Armadilha a evitar:** a chamada ao antifraude acontece **antes** e **fora** do `@Transactional`. Chamar um serviço lento dentro da transação prende a conexão do pool e os locks das contas durante toda a espera. Escreva um teste ou um experimento que mostre o pool esgotando na versão errada.
- **Pronto quando:** derrubar o antifraude não trava a `wallet-api`, o fallback tem uma regra explícita (ex.: recusar acima de X, aceitar abaixo) e o estado do circuito aparece no Actuator.

**Leituras:**
- Resilience4j docs (Spring Boot) — https://resilience4j.readme.io/docs/getting-started-3
- Spring Framework Reference: "REST Clients" — https://docs.spring.io/spring-framework/reference/integration/rest-clients.html

---

### Semana 10: Docker das apps + observabilidade

- **Estudar:** Dockerfile multi-stage (ou Buildpacks com `spring-boot:build-image`), Micrometer, Prometheus, tracing distribuído.
- **Construir:**
  - imagem de cada serviço, com tudo subindo no Compose;
  - Prometheus coletando `/actuator/prometheus`;
  - tracing com Micrometer Tracing + Zipkin seguindo a transferência entre os serviços. Para o trace atravessar o RabbitMQ, habilite a observação no `RabbitTemplate` e no listener container (`observation-enabled`).
- **Pronto quando:** dá para ver no Zipkin uma transferência atravessando os dois serviços.
- **(+)** Grafana com um painel simples (transferências por minuto, latência p95).
- **(+)** Logs estruturados em JSON com o trace id (o Spring Boot já tem suporte nativo a logging estruturado).

**Leituras:**
- Spring Boot Reference: "Observability" e "Tracing" — https://docs.spring.io/spring-boot/reference/actuator/observability.html
- Spring Boot Reference: "Efficient Container Images"

---

### Semana 11: Spring Security + JWT

- **Estudar:** cadeia de filtros (`SecurityFilterChain`), OAuth2 Resource Server, JWT (assinatura, claims, expiração), autorização por método (`@PreAuthorize`).
- **Evitar:** tutoriais com "filtro JWT feito à mão" (`OncePerRequestFilter` parseando o token). É um padrão antigo e o próprio Spring já resolve isso.
- **Construir:** escolha **uma** opção:
  - **(a)** `wallet-api` como resource server (`spring-boot-starter-oauth2-resource-server`) + endpoint de login que emite o token com `NimbusJwtEncoder`;
  - **(b)** **Keycloak** no Compose como servidor de identidade. É mais trabalho, mas o Keycloak aparece bastante em vaga.
  - Nas duas opções, cada cliente só vê e movimenta as próprias contas.
- **Pronto quando:** um teste prova que o cliente A recebe 403 ao tentar ver ou movimentar a conta do cliente B.

**Leituras:**
- Spring Security Reference: "OAuth 2.0 Resource Server — JWT" — https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html
- Keycloak: Getting started (Docker) — https://www.keycloak.org/getting-started/getting-started-docker

---

### Semana 12: Vitrine

- **README** com:
  - diagrama da arquitetura (Mermaid renderiza no GitHub);
  - como rodar tudo com um comando;
  - as decisões tomadas e o porquê: ledger, `BigDecimal`, lock pessimista vs otimista, `Idempotency-Key`, outbox, consumidor idempotente, circuit breaker fora da transação.
- **ADRs curtos** (Architecture Decision Records) em `docs/adr/`, um por decisão importante. Cada um responde: contexto, decisão, alternativas e consequências.
- Seção "camadas vs hexagonal": como o código está organizado e o que mudaria.
- **(+)** Manifestos Kubernetes básicos (Deployment, Service, ConfigMap, Secret) rodando no kind ou no minikube.
- **Pronto quando:** alguém que nunca viu o repo consegue subir tudo e entender o desenho em 5 minutos. É isso que o recrutador técnico vai abrir.

---

### Semanas 13–14: Kafka **ou** Angular (decidido na semana 8)

**Opção A: Kafka** (se as vagas de fintech pedirem mais Kafka que Angular)
- **Estudar:** tópicos, partições, consumer groups, offsets, chave de partição e ordenação, Spring for Apache Kafka, e as diferenças em relação ao RabbitMQ (log vs fila).
- **Construir:** um novo consumidor (ex.: `extrato-analytics` ou auditoria) recebendo os eventos da outbox via Kafka, com a conta como chave de partição para manter a ordem por conta; Kafka no Compose (modo KRaft); teste com Testcontainers Kafka.
- **Pronto quando:** os eventos de uma mesma conta chegam em ordem, e o README explica quando escolher Kafka e quando escolher RabbitMQ.

**Opção B: Angular** (se as vagas pedirem perfil Java/Angular)
- **Estudar:** componentes standalone, serviços e injeção, signals, RxJS básico, `HttpClient`, roteamento, interceptors.
- **Construir:** front com login (interceptor colocando o JWT), extrato e tela de transferência (gerando a `Idempotency-Key` no cliente).
- **Pronto quando:** o fluxo completo funciona pelo navegador. Vindo do React, o mais novo vai ser RxJS e o modelo de DI.

---

## 6. Regras para manter o ritmo

1. **Um PR por semana, com o CI verde**, mesmo que pequeno. O histórico de commits é prova de constância.
2. **Se uma semana atrasar**, corte primeiro o **(+)**, depois o "Estudar", e nunca o "Pronto quando".
3. **Todo "Pronto quando" é provado por um teste**, salvo os critérios visuais (logs, Swagger, Zipkin).
4. **Só entra no CV o que estiver no repo e funcionando.**
5. **Cada semana deixa uma frase no README** respondendo "que problema resolvi e por quê assim". No fim, o material de entrevista já está escrito.

## 7. Histórias para contar em entrevista

- **Concorrência (semana 4):** como o saldo negativo foi evitado com acessos simultâneos, e por que lock pessimista com ordem fixa por id em vez de otimista (ou o contrário).
- **Idempotência da API (semana 5):** o que acontece quando o cliente reenvia a mesma transferência por timeout.
- **Outbox, "antes e depois" (semanas 6–7):** o evento perdido com o RabbitMQ fora do ar e como o outbox resolveu isso, mais o consumidor idempotente para o at-least-once.
- **Resiliência (semana 9):** por que a chamada externa fica fora da transação, e o que o fallback decide.

> **Sobre as leituras:** os links apontam para documentação oficial ou para fontes estáveis (microservices.io, RFC). Os artigos do Baeldung estão só pelo título: busque pelo nome e confira se o exemplo é do Spring Boot 3 ou 4.
