# Prompts utilizados no projeto

Registro exigido pelo README do desafio ("pesquisas, prompts, códigos de referência... anexados ao projeto e comitados").

- **Ferramenta:** Claude Code (extensão VS Code), modelo Claude Sonnet 5.5.
- **Data:** 2026-10-08.
- Os prompts do usuário estão **na íntegra, como foram digitados** (inclusive erros de digitação).
- Os prompts enviados aos subagentes estão em [prompts-agentes.md](prompts-agentes.md).
- Registros por desenvolvedor (o que foi gerado e verificado): `2026-10-08-dev1-coordinator.md`, `…dev2-reserve.md`, `…dev3-merchant.md`, `…dev4-estrutura-contratos-orquestrador.md`.

## Resumo da sequência

| # | Objetivo | Principais resultados |
|---|----------|----------------------|
| 1 | Clonar o repositório | `entrevista_inter/` |
| 2 | Planejar e documentar no Obsidian | vault `docs/obsidian-vault` (notas 00–09) |
| 3 | Manter o Java do projeto | decisão: Java 25 mantido |
| 4 | Ler o diagrama da equipe | comparação diagrama × plano |
| 5 | Atualizar as notas pelo diagrama | notas 02, 03, 04, 06, 09 reescritas |
| 6 | Analisar o último commit | rotas `/reserves`, `/receivables`, status `REJECTED` |
| 7 | Dividir o trabalho em 4 devs | nota 10 |
| 8 | Executar as tarefas do Dev 4 | multi-módulo, contratos, orquestrador, nota 11 |
| 9 | Um agente por dev | Coordinator, Reserve e Merchant implementados, nota 12 |
| 10 | Docker: docs e testes | `Dockerfile`, `docker-compose.yml`, site de docs, nota 13 |
| 11 | Informar o fork | só consulta; nenhuma alteração feita |
| 12 | Documentar os prompts | este arquivo |

---

## Prompt 1 — Clonar o projeto

```text
Faça o clone do projeto https://github.com/TheJacksonLima/entrevista_inter.git
```
**Resultado:** `git clone` em `D:\Empresa\Projetos\inter\entrevista_inter`.

---

## Prompt 2 — Planejamento e documentação no Obsidian

```text
Tenho o seguinte desafio tecnico:

## SPP — Sistema de Pagamento com Porquinhos

Piggie é a nova stablecoin do Inter. O desafio é implementar o Sistema de Pagamento com Porquinhos (SPP).

Um pagamento envolve algumas etapas:

1. Cliente inicia um pagamento por meio de um QR Code que identifica o merchant e a quantidade de Piggies
2. Em paralelo, a intenção de pagamento é registrada, o saldo do cliente é reservado e o merchant é validado
3. Quando as três operações concluem com sucesso, o pagamento é confirmado: Piggies são debitados do cliente, creditados ao merchant e o evento é publicado para outros sistemas

Exemplo:
- Cliente A inicia pagamento de 100 Piggies ao Merchant X
- Intenção de pagamento é registrada
- Reserva de 100 Piggies é criada na conta do Cliente A
- Merchant X é validado como ativo
- Pagamento é confirmado: 100 Piggies debitados de A, 100 creditados a X

**IMPORTANTE** nessa dinâmica vamos implementar uma versão simplificada do sistema:
- Pagamentos são sempre por um valor inteiro de Piggies
- Não há cancelamento após confirmação

Vocês devem implementar 3 serviços:

- **PiggiesPaymentCoordinator** é o ponto de entrada do App, registra a intenção de pagamento e orquestra a confirmação de ponta a ponta.
- **PiggiesReserveService** reserva e confirma o débito do cliente. Persiste o estado da reserva em PostgreSQL.
- **PiggiesMerchantService** valida o merchant e registra o crédito. Persiste recebíveis em PostgreSQL e consome/publica eventos de pagamento via Kafka.

Ele precisa ser feito utilizando java 21 com micronaut. 
Planeje para mim as atividades que precisam ser executadas. Gere a documentação no Obsidian de forma explicativa
```
**Resultado:** leitura do scaffold clonado e criação do vault `docs/obsidian-vault` (notas 00–09: visão geral, arquitetura, contratos, um documento por serviço, testes, cronograma, decisões e riscos).
**Observação:** o scaffold usava Java 25, divergindo do "Java 21" pedido → ver Prompt 3.

---

## Prompt 3 — Versão do Java

```text
Utilize o Java que ja esta configurado no projeto
```
**Resultado:** Java 25 mantido (`pom.xml` e `mise.toml` intocados); notas ajustadas.

---

## Prompt 4 — Leitura do diagrama

```text
Foi criada o arquivo diagrama.md faça a leitura do diagrama.
```
**Resultado:** leitura de `docs/diagramas.md` e comparação com o plano (intenção síncrona, crédito via HTTP, um único evento publicado pelo Merchant, status `PENDING` na reserva).

---

## Prompt 5 — Atualizar as notas

```text
Sim atualize as noas
```
(resposta à pergunta "Quer que eu atualize as notas do vault para seguir o seu diagrama?")
**Resultado:** notas 01–09 alinhadas ao diagrama da equipe.

---

## Prompt 6 — Analisar o último commit

```text
Analise o ultimo commit e atualize as notas
```
**Resultado:** análise do commit `3eccfca`; rotas `POST /reserves`, `POST /reserves/{id}/confirm`, `GET /merchants/{id}/status`, `POST /receivables`; status final `REJECTED`; lacuna registrada (reserva órfã no ramo rejeitado).

---

## Prompt 7 — Divisão para 4 desenvolvedores

```text
Somos 4 desenvolvedores, divida a implementação para que os 4 desenvolvedores possam atuar
```
**Resultado:** nota `10 - Divisão da Equipe (4 devs)` (papéis, interfaces combinadas, linha do tempo, checkpoints, regras de trabalho).

---

## Prompt 8 — Tarefas do Dev 4

```text
Realize as tarefas referente ao Dev 4 para que os outros desenvolvedores possam atuar. Atualize tambem o obisidian com a implementação.
```
**Resultado:** projeto convertido em multi-módulo Maven; `contracts/` (3 OpenAPI + JSON Schema do evento); `testing-support`; infra local; `DefaultPaymentOrchestrator`, gateways HTTP e testes; nota `11 - Implementação Dev 4`.

---

## Prompt 9 — Múltiplos agentes, um por dev

```text
Utilize multiplos agentes, 1 para cada Dev e realize a implementação
```
**Resultado:** três subagentes em paralelo (Dev 1 Coordinator, Dev 2 Reserve, Dev 3 Merchant); fase C do Dev 4 (`scripts/e2e.sh`, contrato do Merchant com 409/503); nota `12 - Implementação Devs 1, 2 e 3`.
**Prompts dos subagentes:** [prompts-agentes.md](prompts-agentes.md).

---

## Prompt 10 — Docker

```text
Suba o projeto em um container docker capaz de visualizar a documentação e realizar testes
```
**Resultado:** `Dockerfile` (tooling/builder/service), `docker-compose.yml` com perfis `app`, `docs`, `test` e `e2e`, Swagger UI (:8088), site de documentação com Mermaid e wikilinks (:8089), nota `13 - Docker`.
**Limitação declarada:** o Docker não respondia na máquina; nenhuma imagem foi construída nem iniciada.

---

## Prompt 11 — Informar o fork

```text
Foi realiado um fork do projeto para o seguite caminho https://github.com/csfinfor-lab/entrevista_inter.git
```
**Resultado:** apenas consulta (`git remote -v`, branch e commits). Nenhum remoto foi alterado e nada foi enviado; foi perguntado se o `origin` deveria ser trocado, adicionado como segundo remoto ou com `push`.

---

## Prompt 12 — Documentar os prompts

```text
Gere uma documentação com todos os prompts utilizado no projeto
```
**Resultado:** este arquivo e [prompts-agentes.md](prompts-agentes.md).

---

## Observações de uso responsável

- Todo código gerado deve ser lido e entendido por quem o commita ("a IA que fez" elimina, segundo o README).
- O que **não** foi verificado em execução (testes com Postgres/Kafka, imagens Docker) está declarado nas notas `12` e `13` do vault e em cada registro por desenvolvedor.
- Não há código ou documento de terceiros copiado: o enunciado vem do repositório do desafio e as bibliotecas são dependências Maven públicas.
