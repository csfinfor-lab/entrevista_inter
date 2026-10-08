---
tags: [spp, docker, infra]
---

# 13 — Docker: serviços, documentação e testes

Voltar: [[00 - Índice]] · Veja também: [[12 - Implementação Devs 1, 2 e 3]]

> [!danger] Não foi executado
> O Docker **não responde na máquina onde isto foi escrito**. Nada abaixo foi construído nem iniciado. Validado de fato: `docker compose config` (sintaxe), o **uber-jar** (`mvn package` + `java -jar`, que sobe e só falha ao conectar no Postgres, como esperado), a sintaxe do `e2e.sh` e a lógica JS do site de documentação (rodada em Node). Tags de imagem usadas e **não confirmadas na prática**: `eclipse-temurin:25-jdk/25-jre`, `swaggerapi/swagger-ui:v5.17.14`, `nginx:1.27-alpine`, `alpine:3.20`. Se algum build falhar, comece pela seção *Solução de problemas*.

## O que existe
| Arquivo | Função |
|---------|--------|
| `Dockerfile` | 3 alvos: `tooling` (JDK 25 + Maven do wrapper), `builder` (empacota os uber-jars) e `service` (runtime enxuto, `--build-arg MODULE=…`, usuário não-root) |
| `docker-compose.yml` | infra + perfis `app`, `docs`, `test`, `e2e` |
| `docker/docs-site/` | `index.html` (Docsify + Mermaid) e `_sidebar.md` |
| `.dockerignore` / `.gitattributes` | contexto de build enxuto; força LF em `mvnw`/`*.sh`/`Dockerfile` |
| `{coordinator,reserve,merchant}/pom.xml` | agora declaram o `maven-shade-plugin` (configuração vem do `micronaut-parent`) para gerar o jar executável |

## Perfis do compose
| Comando | O que sobe |
|---------|-----------|
| `docker compose up -d` | só **Postgres :5432** e **Kafka :9092** (para rodar os serviços com `mn:run`) |
| `… --profile app up -d --build` | + **coordinator :8080**, **reserve :8081**, **merchant :8082** |
| `… --profile docs up -d` | + **Swagger UI :8088** e **site de docs :8089** |
| `… --profile test run --rm tests-unit` | testes que não precisam de Docker |
| `… --profile test run --rm tests` | **todos** os testes (Testcontainers) |
| `… --profile app --profile e2e run --rm e2e` | `scripts/e2e.sh` contra os serviços |

Tudo menos testes: `docker compose --profile app --profile docs up -d --build`.

## Visualizar a documentação
- **Swagger UI — http://localhost:8088**: os 3 contratos OpenAPI (menu *Select a definition*). Com o perfil `app` no ar, o *Try it out* funciona porque os serviços recebem `MICRONAUT_SERVER_CORS_ENABLED=true` (só no compose; não está no `application.yml`).
- **Docs — http://localhost:8089**: README, `docs/diagramas.md`, todo o **vault do Obsidian** e os registros de IA, com barra lateral.
  - **Mermaid** é renderizado no navegador.
  - `[[wikilinks]]` viram links; o front matter é removido; callouts `> [!note]` viram citações com rótulo.
  - Os arquivos são **montados somente-leitura** do repositório: editar uma nota e dar F5 já mostra a mudança.
  - Os scripts (Docsify e Mermaid) vêm de CDN: o **navegador** precisa de internet.
  - Ao criar uma nota nova, inclua-a em `docker/docs-site/_sidebar.md` (o conteúdo aparece pelo link `[[…]]` mesmo sem sidebar).

## Rede e configuração dentro do compose
Os serviços são configurados por variáveis de ambiente (já previstas nos `application.yml`):

| Serviço | Variáveis |
|---------|-----------|
| coordinator | `COORDINATOR_DB_URL=…postgres:5432/coordinator`, `RESERVE_URL=http://reserve:8081`, `MERCHANT_URL=http://merchant:8082` |
| reserve | `RESERVE_DB_URL=…postgres:5432/reserve` |
| merchant | `MERCHANT_DB_URL=…postgres:5432/merchant`, `KAFKA_BOOTSTRAP_SERVERS=kafka:19092` |

**Kafka com dois listeners:** `kafka:19092` (dentro da rede) e `localhost:9092` (host). Sem isso, um cliente dentro do compose receberia `localhost:9092` como endereço do broker e não conectaria.

Os serviços esperam o Postgres ficar *healthy* (`pg_isready`). Não há healthcheck nos serviços Java (a imagem JRE não tem `curl`); o `e2e.sh` aguarda `/health` por até 120 s.

## Como os testes rodam em container
- **`tests-unit`**: `mvn test` excluindo `*ApplicationTest`, `PaymentControllerTest`, `ReserveControllerIntegrationTest` e `MerchantApiIntegrationTest`. Não precisa de socket. Se criar novos testes com Testcontainers, adicione-os à exclusão no compose.
- **`tests`**: `mvn test` completo. Testcontainers precisa do Docker do host, então o compose monta `/var/run/docker.sock` e define `TESTCONTAINERS_HOST_OVERRIDE=host.docker.internal` (+ `extra_hosts … host-gateway`) para o teste alcançar os containers irmãos que ele mesmo cria.
- O repositório é montado em `/workspace` (testes sempre sobre o código atual) e o cache do Maven fica no volume `m2cache`. Relatórios em `*/target/surefire-reports` no host.
- Aqui roda **JDK 25 de verdade**, sem os `-Djdk.version=21` usados no desenvolvimento local — é a primeira validação real do alvo do `pom.xml`.

## Solução de problemas
| Sintoma | Causa provável | Ação |
|---------|----------------|------|
| `exec ./mvnw: no such file` / `^M` | fim de linha CRLF | o Dockerfile já faz `sed`; se vier de outro lugar, `git config core.autocrlf false` e refaça o checkout |
| `manifest unknown` em `eclipse-temurin:25-*` | tag inexistente no registry usado | trocar por outra imagem JDK/JRE 25 (ou `21` + `-Djdk.version=21 -Drelease.version=21`) |
| Serviço cai com `Connection refused` ao Postgres | subiu antes do banco | `docker compose --profile app up -d` de novo (há `depends_on: service_healthy`); ver `docker compose logs postgres` |
| Merchant não publica / `Connection to node … could not be established` | listener do Kafka | confirmar `KAFKA_BOOTSTRAP_SERVERS=kafka:19092` dentro do compose e `localhost:9092` no host |
| `tests` não acha o Docker | socket não montado (Windows: precisa do Docker Desktop com WSL2) | usar `tests-unit`, ou rodar `mvn test` direto no host |
| Banco "antigo" com schema velho | volume `pgdata` persistente | `docker compose down -v` |
| Páginas sem diagrama em :8089 | CDN do Mermaid bloqueada | liberar `cdn.jsdelivr.net` no navegador |

## Para quem for validar
1. `docker compose --profile app --profile docs up -d --build` — observar se os 3 builds terminam.
2. Abrir :8088 e :8089.
3. `docker compose --profile app --profile e2e run --rm e2e` — **primeiro teste real** do fluxo completo (também exercita SQL nativo, Hibernate e Kafka nunca executados antes, ver riscos em [[12 - Implementação Devs 1, 2 e 3]]).
4. `docker compose --profile test run --rm tests`.
