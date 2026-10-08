# Prompts enviados aos subagentes

Contexto: a pedido do usuário ([Prompt 9](prompts.md#prompt-9--múltiplos-agentes-um-por-dev)) a implementação foi dividida em três subagentes executados em paralelo, um por desenvolvedor, cada um dono exclusivo de um módulo. O Dev 4 foi executado diretamente na sessão principal.

Os prompts abaixo foram escritos pelo assistente (Claude Code) e enviados aos subagentes (tipo `general-purpose`). O bloco **Ambiente comum** era idêntico nos três; aqui aparece uma única vez, e cada agente recebeu o seu comando Maven com `-pl <módulo>`.

## Ambiente comum (parte dos três prompts)

```text
## Ambiente (IMPORTANTE)
- A máquina NÃO tem JDK 25 (o pom diz 25 e deve continuar assim). Compile/teste com JDK 21 sobrescrevendo
  na linha de comando. Use exatamente, no bash:
  export JAVA_HOME=/c/Users/claeb/.jdks/ms-21.0.7; export PATH="$JAVA_HOME/bin:$PATH"
  S="<scratchpad>/settings.xml"
  cd /d/Empresa/Projetos/inter/entrevista_inter && mvn -s "$S" -B -Djdk.version=21 -Drelease.version=21 -pl <módulo> test -Dtest='<só testes sem Docker>'
  (o settings.xml é obrigatório: o do usuário aponta para um Nexus inacessível; o pai e testing-support já
  estão instalados no ~/.m2, então use `-pl <módulo>` SEM `-am`. Use `timeout` ~110s por chamada ou rode em
  background se demorar.)
- Docker NÃO está disponível: testes que estendem AbstractPostgres*IntegrationTest não podem rodar aqui.
  Escreva-os com cuidado (devem compilar com `test-compile`) e rode de verdade apenas os que não precisam de
  Docker, excluindo os demais via -Dtest. Diga claramente no resultado final o que rodou e o que não pôde
  rodar. NUNCA afirme que um teste passou sem tê-lo executado.
- Prefira a ferramenta Write para criar arquivos (heredocs no shell costumam falhar).
- Nunca faça git commit/push. Não toque em ~/.m2/settings.xml.
- Registre um arquivo curto em docs/ai/ (prompt resumido, o que foi gerado, o que foi verificado).

## Resultado final (formato)
Lista de arquivos criados/alterados, comandos executados com resultado real (nº de testes, falhas), o que NÃO
foi verificado, e divergências/sugestões sobre contratos.
```

---

## Dev 1 — Coordinator (borda e persistência)

```text
Você é o **Dev 1** de uma equipe de 4 devs implementando o SPP (Sistema de Pagamento com Porquinhos), Java +
Micronaut 4.10.6, Maven multi-módulo em D:\Empresa\Projetos\inter\entrevista_inter. Você é DONO exclusivo do
módulo `coordinator` (porta 8080). NÃO edite outros módulos, o pom pai, `contracts/` nem `docs/obsidian-vault`
(outros devs trabalham em paralelo neles). Se achar um problema nesses arquivos, relate no resultado final.

## Leia primeiro
- contracts/coordinator.openapi.yaml (SEU contrato — fonte da verdade), contracts/reserve.openapi.yaml,
  contracts/merchant.openapi.yaml (só para contexto)
- docs/diagramas.md, docs/obsidian-vault/11 - Implementação Dev 4.md, 04 - PiggiesPaymentCoordinator.md
- Código existente em coordinator/src/main/java/co/inter/piggies/coordinator/ (domain, port, gateway, client,
  config, service) — já pronto e testado (orquestrador do Dev 4). NÃO altere o orquestrador/gateways/clients.

## Sua tarefa (Coordinator — borda e persistência)
1. Entidade JPA `Payment` (pacote ...coordinator.domain) com tabela `payment`: id uuid (= paymentId),
   client_id, merchant_id, amount (long), status (enum PaymentStatus: PENDING, CONFIRMED, REJECTED),
   failure_reason (RejectionReason, nullable), idempotency_key (UNIQUE nullable), created_at/updated_at.
   Repositório Micronaut Data (JPA).
2. POST /payments conforme o contrato: valida body (Bean Validation: clientId/merchantId não vazios,
   amount >= 1) → 400 application/problem+json; header opcional Idempotency-Key (mesma chave → mesmo paymentId,
   sem novo pagamento nem nova orquestração; trate a corrida de duas requisições simultâneas com a constraint
   UNIQUE); grava PENDING de forma síncrona e commitada ANTES de acionar o orquestrador; responde 202 com header
   Location: /payments/{id} e body {paymentId,status}. Depois do commit chama
   PaymentOrchestrator.submit(new PaymentIntent(...)) (não bloqueie a resposta; ignore o future).
3. GET /payments/{paymentId}: 200 com PaymentStatusResponse do contrato (reason só quando REJECTED) ou 404
   problem+json.
4. Implemente PaymentStatusUpdater como @Singleton (markConfirmed / markRejected gravando status/reason/
   updated_at; transacional; idempotente — não regrida CONFIRMED para outra coisa). O stub
   LoggingPaymentStatusUpdater é @Secondary e será ignorado automaticamente.
5. Apague Placeholder.java ao criar a entidade real.
6. Use Lombok se quiser (já no pom). Java records para DTOs com @Serdeable.
7. Testes obrigatórios: unitários (serviço de pagamento, idempotência, updater) e de integração
   (@MicronautTest estendendo co.inter.piggies.testing.AbstractPostgresIntegrationTest, com PaymentOrchestrator
   substituído por um mock/fake via @MockBean/@Replaces — POST→202→GET; 400; 404; Idempotency-Key repetida;
   updater). Teste os endpoints HTTP com @Client("/") HttpClient.

(+ bloco "Ambiente comum" acima)
```

---

## Dev 2 — Reserve

```text
Você é o **Dev 2** de uma equipe de 4 devs implementando o SPP (Sistema de Pagamento com Porquinhos), Java +
Micronaut 4.10.6, Maven multi-módulo em D:\Empresa\Projetos\inter\entrevista_inter. Você é DONO exclusivo do
módulo `reserve` (porta 8081). NÃO edite outros módulos, o pom pai, `contracts/` nem `docs/obsidian-vault`
(outros devs trabalham em paralelo). Se achar um problema nesses arquivos, relate no resultado final.

## Leia primeiro
- contracts/reserve.openapi.yaml (SEU contrato — fonte da verdade)
- docs/diagramas.md, docs/obsidian-vault/05 - PiggiesReserveService.md, 11 - Implementação Dev 4.md
- reserve/ atual: ReserveApplication, Placeholder.java, application.yml, smoke test.

## Sua tarefa (PiggiesReserveService, PostgreSQL)
Dono do saldo em Piggies (valores inteiros, long). Modelo (veja a nota 05):
- account(client_id PK, available bigint CHECK>=0, reserved bigint CHECK>=0);
  reservation(payment_id uuid PK/UNIQUE, client_id, amount, status PENDING|CONFIRMED|RELEASED, created_at,
  updated_at).
- Seed de contas para demo/testes: A=1000, B=50, POOR=0 (idempotente; não sobrescreva saldos existentes).
- Endpoints exatamente como o contrato: POST /reserves (201 nova / 200 já existente p/ o mesmo paymentId —
  idempotente / 409 saldo insuficiente / 400 body inválido; conta inexistente: use 409 problem+json),
  POST /reserves/{paymentId}/confirm (PENDING→CONFIRMED; já CONFIRMED→200; RELEASED→409; inexistente→404),
  POST /reserves/{paymentId}/release (PENDING→RELEASED devolvendo ao available; já RELEASED→200;
  CONFIRMED→409; inexistente→404), GET /reserves/{paymentId}. Erros em application/problem+json.
- Semântica: reservar = available -= v, reserved += v; confirmar = reserved -= v (debitado);
  liberar = reserved -= v, available += v.
- Atomicidade sob concorrência: reservar com um único
  UPDATE account SET available = available - :v, reserved = reserved + :v WHERE client_id = :c AND available >= :v
  (0 linhas → saldo insuficiente), na mesma transação do INSERT da reservation. Trate a corrida de dois
  requests com o mesmo paymentId (UNIQUE/PK → devolva a reserva existente com 200, sem debitar duas vezes).
  Confirm/release também devem ser seguros sob concorrência.
- Apague Placeholder.java ao criar as entidades reais. Lombok disponível; DTOs como records @Serdeable;
  Bean Validation (amount >= 1, ids não vazios).
- Testes obrigatórios: unitários (regras de transição/idempotência no serviço — projete o serviço de modo que
  a lógica de decisão seja testável sem banco quando possível) e integração (@MicronautTest estendendo
  AbstractPostgresIntegrationTest; @Client("/") HttpClient): fluxo feliz com saldos corretos, idempotência,
  409 saldo insuficiente, confirm/release e conflitos 409, 404, e um teste de concorrência (20 threads
  reservando 10 cada, em conta de 100 → exatamente 10 sucessos, available=0, reserved=100).
- Como a lógica SQL não pôde ser executada, revise-a com extremo cuidado (sintaxe, nomes de colunas,
  transações) e descreva os riscos.

(+ bloco "Ambiente comum" acima)
```

---

## Dev 3 — Merchant e Kafka

```text
Você é o **Dev 3** de uma equipe de 4 devs implementando o SPP (Sistema de Pagamento com Porquinhos), Java +
Micronaut 4.10.6, Maven multi-módulo em D:\Empresa\Projetos\inter\entrevista_inter. Você é DONO exclusivo do
módulo `merchant` (porta 8082). NÃO edite outros módulos, o pom pai, `contracts/` nem `docs/obsidian-vault`
(outros devs trabalham em paralelo). Se achar um problema nesses arquivos, relate no resultado final.

## Leia primeiro
- contracts/merchant.openapi.yaml e contracts/events/pagamento-confirmado.v1.schema.json (SEUS contratos)
- docs/diagramas.md, docs/obsidian-vault/06 - PiggiesMerchantService.md, 11 - Implementação Dev 4.md
- merchant/ atual: MerchantApplication, Placeholder.java, application.yml (já tem kafka.bootstrap.servers e
  spp.topics.pagamento-confirmado), smoke test; testing-support (ContractSchemas.assertValid,
  AbstractPostgresKafkaIntegrationTest).

## Sua tarefa (PiggiesMerchantService, PostgreSQL + Kafka)
- Entidades: merchant(id varchar PK, name, status ACTIVE|INACTIVE); receivable(id uuid PK, payment_id uuid
  UNIQUE, merchant_id FK, amount bigint, created_at, event_published boolean). Seed idempotente na subida:
  merchant X ACTIVE, Y INACTIVE.
- GET /merchants/{merchantId}/status → 200 {merchantId, active} ou 404 problem+json.
- POST /receivables {paymentId, merchantId, amount}: 404 merchant desconhecido, 409 merchant inativo, 400 body
  inválido; idempotente por paymentId: novo → grava recebível e publica o evento, responde 201; já existente
  → 200 com o mesmo recebível. Trate a corrida de duas requisições com o mesmo paymentId.
- Evento: publique PagamentoConfirmado no tópico de spp.topics.pagamento-confirmado
  (piggies.pagamento-confirmado.v1), chave = paymentId, payload exatamente conforme o JSON Schema (eventId
  uuid novo, paymentId, merchantId, amount, occurredAt ISO-8601 UTC; sem campos extras). Use @KafkaClient
  (acks=all). Publique depois do commit do recebível; marque event_published=true ao confirmar a publicação;
  num retry do mesmo paymentId com event_published=false, republique. O evento nunca deve existir sem o
  recebível gravado.
- GET /merchants/{merchantId}/receivables → lista (404 se merchant desconhecido).
- Apague Placeholder.java. Lombok disponível; DTOs records @Serdeable; Bean Validation; erros em
  application/problem+json.
- Testes obrigatórios: unitários (serviço de recebíveis: cria, duplicata idempotente, merchant inexistente/
  inativo, republicação quando event_published=false; publisher atrás de uma interface para testar sem Kafka;
  o JSON serializado do evento deve validar contra o schema com ContractSchemas.assertValid) e integração
  (@MicronautTest estendendo AbstractPostgresKafkaIntegrationTest): GET status (X, Y, Z 404), POST /receivables
  201/200/409/404/400, consumir o tópico com um consumer do teste, validar a mensagem contra o schema e chave =
  paymentId; mesmo paymentId 2x → 1 recebível e 1 evento.

(+ bloco "Ambiente comum" acima)
```

---

## Dev 4 — Plataforma, contratos e orquestração

Executado diretamente na sessão principal (sem subagente), a partir do [Prompt 8](prompts.md#prompt-8--tarefas-do-dev-4) e do plano da nota `10 - Divisão da Equipe (4 devs)` do vault. Não há prompt separado.
