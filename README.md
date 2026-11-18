# Comando

Aplicação web para **gestão de demandas corporativas**: um colaborador solicita uma demanda a outro
colaborador ou setor (o próprio ou outro), e o atendimento, a comunicação, os arquivos e o acompanhamento
ficam centralizados.

- Frontend React em `http://localhost:8080`
- API REST em `http://localhost:8080/api/v1` (documentação Swagger em `http://localhost:8080/docs/`)
- Instalação somente com **Docker** e **Docker Compose** (nenhum PHP, Composer, Node.js, npm ou PostgreSQL na máquina)

Sumário: [Funcionalidades](#funcionalidades) · [Versões](#versões-utilizadas) · [Estrutura](#estrutura-do-repositório) ·
[Instalação](#instalação) · [Operação diária](#operação-diária) · [Testes](#testes) · [Backup e restauração](#backup-e-restauração) ·
[Atualização](#atualização) · [Banco externo (RDS)](#banco-externo-amazon-rds) · [Desenvolvimento](#desenvolvimento) ·
[Segurança](#segurança) · [Limitações](#limitações-e-itens-não-validados)

---

## Funcionalidades

- **Perfis:** colaborador, gestor (um setor pode ter vários) e administrador. O menu mostra apenas o que o perfil pode usar; o backend valida tudo novamente.
- **Demandas** com protocolo único (`CMD-000001`), categoria, prioridade, prazo desejado (solicitante) e prazo de atendimento (responsável/gestor), anexos, setor de origem histórico e setor de destino.
- **Direcionamento** ao setor (fila sem responsável) ou diretamente a um colaborador ativo do setor escolhido; dentro do próprio setor ou para outro setor.
- **Distribuição:** gestor atribui/reatribui; colaborador assume da fila do setor (operação atômica); administrador transfere entre setores com justificativa.
- **Fluxo:** Aberta → Em andamento ⇄ Aguardando solicitante → Resolvida → Concluída (ou rejeição de volta para Em andamento); cancelamento com justificativa. Concluídas/canceladas são imutáveis.
- **Comentários e anexos privados** (PDF, JPG, JPEG, PNG, WEBP, TXT, DOCX, XLSX; até 5 arquivos de 10 MB), download sempre autorizado no servidor.
- **Histórico imutável** com autor, data, ação, justificativa e valores anteriores/novos.
- **Notificações internas** persistidas (contador, não lidas, marcar uma/todas), sem conteúdo sensível na prévia.
- **Indicadores** na página inicial e listagens com filtros, ordenação e paginação no servidor.
- **Administração** de usuários, setores e categorias (sem exclusão física; inativação).

Telas: Login, Início, Minhas solicitações, Meu atendimento (atribuídas a mim + fila do setor), Demandas do setor (gestor),
Todas as demandas (administrador), Nova demanda, Detalhes da demanda, Notificações, Meu perfil, Usuários, Setores e Categorias.

As decisões técnicas estão em [docs/decisoes.md](docs/decisoes.md).

## Versões utilizadas

| Componente | Versão | Onde |
|---|---|---|
| PHP (FPM, Alpine) | 8.4.15 | `php:8.4.15-fpm-alpine3.22` |
| Laravel | 12.72.0 | `backend/composer.lock` |
| Laravel Sanctum | 4.4.0 | `backend/composer.lock` |
| PHPUnit | 11.5.57 | `backend/composer.lock` |
| Composer | 2.8.12 | `composer:2.8.12` (somente no build) |
| PostgreSQL | 17.9 | `postgres:17.9-alpine3.22` |
| Node.js | 22.14.0 | `node:22.14.0-alpine` (somente no build/dev) |
| React / React DOM | 19.3.1 | `frontend/package-lock.json` |
| React Router | 7.18.5 | `frontend/package-lock.json` |
| Vite | 7.3.8 | `frontend/package-lock.json` |
| Axios | 1.21.1 | `frontend/package-lock.json` |
| Swagger UI | 5.34.1 | `swagger-ui-dist` |
| Nginx (sem privilégios) | 1.28.0 | `nginxinc/nginx-unprivileged:1.28.0-alpine3.21` |
| Alpine (backup de volumes) | 3.22 | `alpine:3.22` |

Validado com Docker Engine 29.4.1 e Docker Compose v5.1.3 (Docker Desktop, Windows 11). Requer Docker Compose v2.24 ou superior.

## Estrutura do repositório

```
comando/
├── backend/              Laravel (API, regras, testes) + Dockerfile (backend/worker/scheduler)
├── frontend/             React + Vite + Dockerfile (build → Nginx)
├── docker/
│   ├── nginx/            default.conf do serviço web
│   ├── scripts/          key.sh / key.ps1 (geração da APP_KEY)
│   └── certs/            certificados de autoridade (ex.: RDS), montados somente leitura
├── docs/                 openapi.json, gerador, decisões técnicas, guia do RDS
├── compose.yaml          instalação (web, backend, worker, scheduler)
├── compose.local-db.yaml complemento: PostgreSQL local
├── compose.dev.yaml      complemento: desenvolvimento (código montado, Vite, porta do banco)
├── compose.test.yaml     testes isolados (PostgreSQL descartável)
├── .env.example
└── README.md
```

Serviços: `web` (Nginx: frontend + proxy da API), `backend` (PHP-FPM), `worker` (fila `database`),
`scheduler` (agendador), `postgres` (somente com `compose.local-db.yaml`) e `frontend-dev` (somente desenvolvimento).

---

## Instalação

Pré-requisitos: **Docker** e **Docker Compose**. Nada mais.

- **Windows:** Docker Desktop (backend WSL 2). Use o PowerShell na pasta do projeto.
- **Linux:** Docker Engine + plugin Compose (`docker compose version` deve responder). Se o usuário não estiver no grupo `docker`, prefixe os comandos com `sudo`.

> Nos exemplos, `$C` abrevia os arquivos Compose. Com banco local:
> `docker compose -f compose.yaml -f compose.local-db.yaml`. Com banco externo: `docker compose -f compose.yaml`.
> Os comandos abaixo aparecem por extenso para poderem ser copiados.

### 1. Criar o `.env`

Linux:

```bash
cp .env.example .env
chmod 600 .env
```

Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

Edite o `.env` e ajuste no mínimo:

- `DB_PASSWORD` — senha forte do banco (usada também para criar o banco local).
- `APP_URL` — endereço que os usuários vão usar, ex.: `http://comando.empresa.local:8080`.
- `SANCTUM_STATEFUL_DOMAINS` — todos os `host:porta` pelos quais a aplicação será acessada no navegador (o host de `APP_URL` já é incluído automaticamente). Ex.: `comando.empresa.local:8080,10.0.0.15:8080`.
- `SESSION_SECURE_COOKIE=true` se o acesso for por HTTPS.
- `WEB_PORT` se a porta 8080 já estiver em uso.

### 2. Gerar a `APP_KEY` (uma única vez)

O script gera a chave **dentro de um container** e a grava no `.env`. Se já houver uma chave, ele não a altera.

Linux:

```bash
sh docker/scripts/key.sh
```

Windows PowerShell:

```powershell
powershell -ExecutionPolicy Bypass -File docker\scripts\key.ps1
```

**Guarde uma cópia segura do `.env`.** A `APP_KEY` não pode ser regenerada a cada reinício: sem ela as sessões e dados cifrados são perdidos.

### 3. Subir a aplicação (banco local)

```bash
docker compose -f compose.yaml -f compose.local-db.yaml up -d --build
```

Para banco externo (RDS), veja [Banco externo](#banco-externo-amazon-rds).

### 4. Configuração inicial e primeiro administrador

```bash
docker compose -f compose.yaml -f compose.local-db.yaml run --rm backend php artisan comando:install
```

O comando verifica a conexão, executa as migrations, cria os cadastros básicos (setor "Administração" e categoria "Geral")
e pede **nome, e-mail e senha** do primeiro administrador. Pode ser executado de novo a qualquer momento: não duplica nem apaga
dados e não altera administradores existentes. As migrations **não** rodam automaticamente na inicialização dos containers.

Uso não interativo (ex.: automação): defina a variável `COMANDO_ADMIN_PASSWORD` no ambiente do terminal (ou do sistema de
automação) e repasse-a ao container com `-e`:

```bash
docker compose -f compose.yaml -f compose.local-db.yaml run --rm -e COMANDO_ADMIN_PASSWORD backend php artisan comando:install --nome="Fulano" --email=admin@empresa.com.br
```

### 5. Acessar

| O quê | Endereço |
|---|---|
| Aplicação | http://localhost:8080 |
| API | http://localhost:8080/api/v1 |
| Documentação (Swagger) | http://localhost:8080/docs/ |
| Saúde | http://localhost:8080/health |

Entre com o administrador criado e cadastre setores, categorias e usuários em **Administração**.

### Dados de demonstração (opcional, somente ambiente local)

Carrega 1 administrador, 2 gestores (Tecnologia e Financeiro), 5 colaboradores, setores, categorias e demandas
entre setores e no mesmo setor, em todos os status. Não apaga nem altera dados existentes; pode ser repetido.
Exige `APP_ENV=local` (recusado em produção):

```bash
docker compose -f compose.yaml -f compose.local-db.yaml run --rm -e APP_ENV=local backend php artisan comando:demo
```

O comando pede a senha dos novos usuários de demonstração (todos com e-mail `@comando.local`, ex.: `ana@comando.local`,
`gestora.ti@comando.local`, `admin@comando.local`).

---

## Operação diária

| Ação | Comando |
|---|---|
| Iniciar | `docker compose -f compose.yaml -f compose.local-db.yaml up -d` |
| Situação/saúde | `docker compose -f compose.yaml -f compose.local-db.yaml ps` |
| Logs (todos, seguindo) | `docker compose -f compose.yaml -f compose.local-db.yaml logs -f --tail=200` |
| Logs de um serviço | `docker compose -f compose.yaml -f compose.local-db.yaml logs -f --tail=200 backend` |
| Reiniciar | `docker compose -f compose.yaml -f compose.local-db.yaml restart` |
| **Parar** (mantém containers e dados) | `docker compose -f compose.yaml -f compose.local-db.yaml stop` |
| **Remover containers** (mantém volumes/dados) | `docker compose -f compose.yaml -f compose.local-db.yaml down` |

> ⚠️ **Nunca** use `down -v`, `docker volume rm` ou `php artisan migrate:fresh` na instalação: esses comandos **apagam o banco e os anexos**.
> Nenhum procedimento deste README precisa deles. Recriar containers (`down` + `up`, ou `up --build`) preserva os dados, que ficam nos volumes
> `comando_postgres`, `comando_anexos` e `comando_storage`.

Os logs vão para a saída dos containers (`docker compose logs`), com rotação (5 arquivos de 10 MB por serviço).
Senhas, cookies e tokens não são registrados.

Dica: para não repetir `-f`, defina no `.env` `COMPOSE_FILE=compose.yaml:compose.local-db.yaml` (Linux) ou
`COMPOSE_FILE=compose.yaml;compose.local-db.yaml` (Windows) e use apenas `docker compose ...`.

---

## Testes

Os testes rodam em um projeto Compose separado (`comando-test`) com PostgreSQL próprio em memória (tmpfs).
Não usam o `.env`, a rede nem os volumes da instalação, e o código de teste recusa qualquer banco que não seja o isolado.

```bash
docker compose -f compose.test.yaml run --rm --build test
docker compose -f compose.test.yaml run --rm --build frontend-build
docker compose -f compose.test.yaml down -v
```

O último comando remove apenas os recursos do projeto **comando-test** (descartáveis).

Cobertura: login/logout/usuário inativo/limite de tentativas, permissões por perfil e participação, abertura no mesmo e em outro setor,
validação do responsável, atribuição concorrente (dois processos reais), protocolos simultâneos, transferências, matriz completa de
transições, resolução/confirmação/rejeição, cancelamento, proteção e validação de anexos (inclusive conteúdo falsificado e limpeza em falhas),
restrições de listagens e indicadores, histórico, notificações (sem duplicidade e sem o autor), último administrador, revogação de sessões,
instalação idempotente e um ciclo completo pela API. Resultado atual: **87 testes, 454 asserções, todos aprovados**.

Validar os arquivos Compose:

```bash
docker compose -f compose.yaml config -q
docker compose -f compose.yaml -f compose.local-db.yaml config -q
docker compose -f compose.yaml -f compose.local-db.yaml -f compose.dev.yaml config -q
```

---

## Backup e restauração

Crie a pasta `backups` na raiz do projeto (já ignorada pelo Git). Guarde os backups fora do servidor.

### Banco de dados (PostgreSQL local)

Backup (formato custom, gerado dentro do container e copiado para o host):

```bash
docker compose -f compose.yaml -f compose.local-db.yaml exec -T postgres sh -c 'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc -f /tmp/comando.dump'
docker compose -f compose.yaml -f compose.local-db.yaml cp postgres:/tmp/comando.dump ./backups/comando-AAAA-MM-DD.dump
```

Restauração (**substitui** o conteúdo atual do banco; pare a aplicação antes):

```bash
docker compose -f compose.yaml -f compose.local-db.yaml stop web backend worker scheduler
docker compose -f compose.yaml -f compose.local-db.yaml cp ./backups/comando-AAAA-MM-DD.dump postgres:/tmp/restaurar.dump
docker compose -f compose.yaml -f compose.local-db.yaml exec -T postgres sh -c 'pg_restore -U "$POSTGRES_USER" -d "$POSTGRES_DB" --clean --if-exists --no-owner --single-transaction /tmp/restaurar.dump'
docker compose -f compose.yaml -f compose.local-db.yaml start backend worker scheduler web
```

Com RDS, use snapshots do RDS ou o dump lógico descrito em [docs/rds.md](docs/rds.md#6-backups-com-rds).

### Anexos

Backup do volume `comando_anexos` (somente leitura durante a cópia):

Linux:

```bash
docker run --rm -v comando_anexos:/dados:ro -v "$PWD/backups:/backup" alpine:3.22 tar czf /backup/anexos-AAAA-MM-DD.tar.gz -C /dados .
```

Windows PowerShell:

```powershell
docker run --rm -v comando_anexos:/dados:ro -v "${PWD}/backups:/backup" alpine:3.22 tar czf /backup/anexos-AAAA-MM-DD.tar.gz -C /dados .
```

Restauração (extrai sobre o volume preservando dono e permissões; troque `$PWD` por `${PWD}` no PowerShell):

```bash
docker run --rm -v comando_anexos:/dados -v "$PWD/backups:/backup:ro" alpine:3.22 tar xzf /backup/anexos-AAAA-MM-DD.tar.gz -C /dados
```

Faça o backup do banco e dos anexos no mesmo momento (de preferência com a aplicação parada) para que fiquem consistentes.
O nome dos volumes segue `COMPOSE_PROJECT_NAME` (padrão `comando`): confira com `docker volume ls`.

### `.env` e `APP_KEY`

O `.env` contém a `APP_KEY` e as senhas: copie-o para um cofre/local seguro sempre que for alterado.
Na restauração em outro servidor, restaure **o mesmo `.env`** (mesma `APP_KEY`) junto com o banco e os anexos.
Nunca execute `php artisan key:generate` em uma instalação existente (o script `key.sh`/`key.ps1` se recusa a substituir uma chave).

---

## Atualização

1. **Backup** do banco, dos anexos e do `.env` (seção anterior).
2. Obtenha a nova versão do código (ex.: `git pull` ou cópia do pacote), preservando o `.env` e a pasta `docker/certs`.
3. Gere as novas imagens (com a aplicação ainda no ar):

```bash
docker compose -f compose.yaml -f compose.local-db.yaml build --pull
```

4. Pare o processamento em segundo plano e aplique as migrations de forma controlada, com a imagem nova:

```bash
docker compose -f compose.yaml -f compose.local-db.yaml stop worker scheduler
docker compose -f compose.yaml -f compose.local-db.yaml run --rm backend php artisan migrate:status
docker compose -f compose.yaml -f compose.local-db.yaml run --rm backend php artisan migrate --force
```

5. Recrie os containers com as imagens novas (os volumes são preservados):

```bash
docker compose -f compose.yaml -f compose.local-db.yaml up -d
docker compose -f compose.yaml -f compose.local-db.yaml ps
```

Para manter imagens de versões anteriores (e voltar rapidamente), defina `COMANDO_TAG` no `.env` antes do build (ex.: `COMANDO_TAG=1.1.0`).
Para voltar uma versão, restaure o código e a `COMANDO_TAG` anteriores e, se a migration nova não for compatível, o backup do banco.

---

## Banco externo (Amazon RDS)

Resumo — detalhes, rede, TLS e migração de dados em **[docs/rds.md](docs/rds.md)**:

1. Coloque o bundle de CA do RDS em `docker/certs/global-bundle.pem`.
2. No `.env`:

```dotenv
DB_HOST=SEU-ENDPOINT.rds.amazonaws.com
DB_PORT=5432
DB_DATABASE=comando
DB_USERNAME=comando
DB_PASSWORD=...
DB_SSLMODE=verify-full
DB_SSLROOTCERT=/etc/comando/certs/global-bundle.pem
```

3. Suba **sem** o arquivo do banco local e instale:

```bash
docker compose -f compose.yaml up -d --build
docker compose -f compose.yaml run --rm backend php artisan comando:install
```

Backend, worker e scheduler recebem a mesma configuração. Não é necessário superusuário nem extensões.

---

## Desenvolvimento

```bash
docker compose -f compose.yaml -f compose.local-db.yaml -f compose.dev.yaml up -d --build
```

- Frontend com recarregamento: http://localhost:5173 (Vite; `/api` e `/sanctum` encaminhados ao Nginx).
- Código de `backend/` e `frontend/` montado nos containers; o backend usa a etapa `development` da imagem.
- PostgreSQL exposto apenas em `127.0.0.1:5433` (`DB_DEV_PORT`).
- Gerar a documentação OpenAPI após mudar a API (sem Python local):

```bash
docker run --rm -v "$PWD/docs:/docs" python:3.13-alpine python /docs/generate-openapi.py
```

O `compose.dev.yaml` depende do `compose.local-db.yaml` (adiciona a porta do PostgreSQL local).

---

## Segurança

- Sessão Sanctum (SPA) em cookie HttpOnly, `SameSite=Lax`, cifrado; `Secure` com `SESSION_SECURE_COOKIE=true` (HTTPS). Expiração por inatividade em `SESSION_LIFETIME` (minutos).
- CSRF em todas as mutações (cookie `XSRF-TOKEN` + cabeçalho `X-XSRF-TOKEN`). Nada de credenciais ou sessão em `localStorage`.
- Logout, inativação e troca de senha revogam sessões; usuários inativos não autenticam.
- Limite de tentativas de login (5 por e-mail+IP por minuto, 20 por IP por minuto) e limite geral da API.
- Senhas com hash bcrypt; mínimo de 10 caracteres com letras e números.
- Policies, Form Requests e API Resources; campos controlados pelo servidor nunca vêm do cliente.
- Nginx com CSP restritiva, `nosniff`, `X-Frame-Options` e sem versão exposta. Comentários exibidos como texto (sem HTML).
- Anexos privados, nomes internos aleatórios, download autorizado a cada requisição.
- Imagens sem segredos (`.env` fora do contexto de build), processos sem root, backend e banco apenas na rede interna.
- CORS desligado por padrão (mesma origem); `CORS_ALLOWED_ORIGINS` só para desenvolvimento.

Para HTTPS, publique a aplicação atrás de um proxy/balanceador com TLS que envie `X-Forwarded-Proto: https`, e ajuste
`APP_URL=https://...` e `SESSION_SECURE_COOKIE=true`. Se o proxy estiver em outra máquina, restrinja `WEB_BIND`/firewall para que
a porta 8080 só aceite conexões do proxy.

---

## Limitações e itens não validados

- **Amazon RDS:** a conexão com RDS e o modo `verify-full` não foram testados contra uma instância real (nenhum recurso AWS foi criado). A configuração usa os parâmetros padrão do driver PostgreSQL/libpq; valide em homologação seguindo [docs/rds.md](docs/rds.md).
- **Linux:** a instalação foi validada em Windows 11 com Docker Desktop. Os comandos para Linux estão documentados, mas não foram executados em um host Linux.
- **Interface:** login, painel, listas, detalhes, rejeição de solução, atribuição, perfil, Swagger e layout de celular foram verificados no navegador; o envio de anexos foi validado pela API (sessão + CSRF reais), não pelo seletor de arquivos do navegador.
- Não há testes automatizados de interface (o frontend é validado pelo build de produção).
- Fora do escopo desta versão: múltiplas empresas, e-mails, WhatsApp, chat em tempo real/WebSocket, reabertura de demandas encerradas, edição de título/descrição/categoria, edição/exclusão de comentários, comentários privados e integração com S3 (a abstração de armazenamento está pronta via `ANEXOS_DISK`).

<!-- Teste de entrega CI-CD por SHA -->
