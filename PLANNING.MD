# ImobClone — Planejamento

Projeto fullstack de escopo controlado com processo profissional completo: TDD, Git flow com PRs, CI/CD, Docker e documentação. Objetivo: praticar todas as camadas exigidas em uma vaga fullstack web.

Domínio: plataforma de aluguel de imóveis (inspirada em QuintoAndar / Zap Imóveis), recortada para um MVP viável.

---

## 1. Visão

Anunciantes cadastram imóveis para aluguel. Buscadores procuram com filtros, favoritam e agendam visitas. O anunciante confirma ou recusa as visitas recebidas.

**MVP (V1) — nada além disso:**

1. Cadastro de usuário
2. Login (JWT)
3. CRUD de imóveis (com regra de dono)
4. Busca de imóveis com filtros, paginação e ordenação
5. Favoritos
6. Agendamento de visitas (criar, confirmar, recusar, cancelar)
7. Frontend React consumindo a API

**Explicitamente fora do V1 (vira issue com label `v2`):**

- Upload de fotos (storage, resize, validação de arquivo) — no V1 o imóvel tem apenas um campo opcional `photoUrls` de texto
- Mapa e geolocalização
- Chat entre usuários
- Avaliações e notas
- Contrato e pagamento
- Recuperação de senha / e-mail de confirmação
- Painel de admin

---

## 2. Stack

| Camada | Tecnologia |
|---|---|
| Backend | NestJS + TypeScript (strict) |
| ORM | Prisma |
| Banco | PostgreSQL (Docker) |
| Testes | Jest + Supertest |
| Validação | class-validator + class-transformer |
| Docs da API | @nestjs/swagger |
| Frontend | React + Vite + TypeScript |
| Containers | Docker + Docker Compose |
| CI | GitHub Actions |
| Deploy | Railway ou Render |

---

## 3. Papéis

Um usuário pode atuar como:

- **ANNOUNCER** — cadastra e gerencia imóveis, confirma/recusa visitas recebidas
- **SEEKER** — busca, favorita, agenda e cancela visitas

Um mesmo usuário pode acumular os dois papéis. Isso obriga autorização real: além de "está autenticado?", checar "este recurso é seu?" e "seu papel permite esta ação?".

---

## 4. Modelo de domínio

| Entidade | Campos | Relacionamento |
|---|---|---|
| **User** | id, name, email, passwordHash, roles (ANNOUNCER \| SEEKER) | tem muitos Property, Visit e Favorite |
| **Property** | id, title, description, type (APARTMENT, HOUSE, STUDIO), street, city, state, zipCode, bedrooms, bathrooms, areaM2, rentPrice, condoFee, iptu, status (AVAILABLE, RENTED, PAUSED), photoUrls (opcional) | pertence a um User (owner) |
| **Visit** | id, scheduledAt, durationMin, status (PENDING, CONFIRMED, CANCELLED, REJECTED, DONE) | pertence a um Property e a um User (seeker) |
| **Favorite** | id, createdAt | pertence a um Property e a um User (seeker); par único |

**Campo derivado:** `totalCost` = rentPrice + condoFee + iptu. Calculado no service, retornado pela API, **nunca salvo no banco**.

---

## 5. Regras de negócio (casos de teste do TDD)

**Imóveis**
1. Apenas o dono pode editar, pausar ou remover o próprio imóvel
2. Imóvel com status RENTED ou PAUSED não aparece na busca pública
3. Preço, área, quartos e banheiros devem ser maiores que zero
4. Custo total (aluguel + condomínio + IPTU) é calculado e retornado; nunca persistido

**Busca**

5. Busca pública retorna apenas imóveis AVAILABLE
6. Filtros combináveis: cidade, tipo, preço min/max, quartos mínimos
7. Resultado sempre paginado (limite máximo por página) e ordenável (preço, data)

**Favoritos**

8. Um usuário não pode favoritar o mesmo imóvel duas vezes
9. Usuário só enxerga e remove os próprios favoritos

**Visitas**

10. Imóvel RENTED ou PAUSED não aceita agendamento
11. Visita não pode colidir com outra visita CONFIRMED no mesmo imóvel (sobreposição de intervalo de tempo)
12. Agendamento exige antecedência mínima de 24h
13. Agendamento só dentro da janela permitida (08:00–20:00)
14. Um seeker não pode ter duas visitas no mesmo horário, mesmo em imóveis diferentes
15. Dono não pode agendar visita no próprio imóvel
16. Apenas o dono do imóvel confirma ou recusa uma visita
17. Apenas o seeker cancela a própria visita, e só até 12h antes do horário marcado

---

## 6. Contrato da API

**Auth**

| Método | Rota | Descrição |
|---|---|---|
| POST | /auth/register | Cadastro |
| POST | /auth/login | Retorna JWT |
| GET | /auth/me | Dados do usuário logado |

**Imóveis**

| Método | Rota | Descrição |
|---|---|---|
| POST | /properties | Cria imóvel (autenticado) |
| GET | /properties?city=&type=&minPrice=&maxPrice=&bedrooms=&sort=&page=&limit= | Busca pública (só AVAILABLE) |
| GET | /properties/:id | Detalhe público |
| GET | /properties/mine | Imóveis do usuário logado (todos os status) |
| PATCH | /properties/:id | Edita (só dono) |
| PATCH | /properties/:id/status | Muda status (só dono) |
| DELETE | /properties/:id | Remove (só dono) |

**Favoritos**

| Método | Rota | Descrição |
|---|---|---|
| POST | /favorites | Favorita um imóvel |
| GET | /favorites | Lista favoritos do usuário |
| DELETE | /favorites/:propertyId | Desfavorita |

**Visitas**

| Método | Rota | Descrição |
|---|---|---|
| POST | /visits | Agenda visita (seeker) |
| GET | /visits/mine | Visitas que o usuário agendou |
| GET | /visits/received | Visitas nos imóveis do usuário (anunciante) |
| PATCH | /visits/:id/confirm | Confirma (só dono do imóvel) |
| PATCH | /visits/:id/reject | Recusa (só dono do imóvel) |
| PATCH | /visits/:id/cancel | Cancela (só o seeker) |

**Códigos:** 201 criação, 400 validação, 401 sem token, 403 sem permissão sobre o recurso, 404 não encontrado, 409 conflito (horário ocupado, favorito duplicado).

---

## 7. Roadmap (cada linha = issue + branch + PR)

| # | Issue | Branch | Pratica |
|---|---|---|---|
| 0 | Setup: Nest + Docker Compose + Prisma + CI verde + main protegida | chore/project-setup | Docker, CI, config |
| 1 | Cadastro e login (JWT, guard, hash) | feat/auth | bcrypt, JWT, guards |
| 2 | CRUD de imóveis + regra de dono | feat/properties-crud | TDD (regras 1–4), autorização |
| 3 | Busca com filtros, paginação e ordenação | feat/properties-search | TDD (regras 5–7), queries |
| 4 | Favoritos | feat/favorites | TDD (regras 8–9), constraint única |
| 5a | Visitas: agendar e listar | feat/visits-schedule | TDD (regras 10–15), colisão de horário |
| 5b | Visitas: confirmar, recusar, cancelar | feat/visits-manage | TDD (regras 16–17), máquina de estados |
| 6 | Swagger + README + ADRs | docs/api-documentation | Documentação |
| 7 | Testes e2e das rotas principais | test/e2e-coverage | Supertest |
| 8 | Deploy | chore/deploy | CD |
| 9 | Frontend React | feat/frontend-* | React, auth no client |

**Critério de pronto da issue 0:**

- `docker compose up -d` sobe o Postgres
- `npm run start:dev` sobe a API
- `npm test` passa
- `.env` fora do git
- CI verde no PR
- Push direto na main é bloqueado

**Issue mais difícil:** 5a. A regra 11 (colisão de intervalos) e as regras de tempo exigem testes com data controlada — usar fake timers do Jest, não `new Date()` solto no service. Injetar um provider de data é a decisão de design correta aqui e rende um bom ADR.

---

## 8. Fluxo de trabalho por feature

1. Abrir issue com descrição e critérios de aceite
2. Criar branch a partir da main (`feat/`, `fix/`, `chore/`, `test/`, `docs/`)
3. Escrever o teste da regra de negócio (falha)
4. Implementar até passar
5. Refatorar
6. Abrir PR linkando a issue (`Closes #N`), descrever o que mudou
7. CI verde, revisar o próprio diff, **squash and merge**

Commits seguem Conventional Commits: `feat:`, `fix:`, `chore:`, `test:`, `docs:`, `refactor:`.

---

## 9. Definição de pronto (por feature)

- [ ] Testes das regras de negócio passando
- [ ] Lint sem erros
- [ ] CI verde no PR
- [ ] PR revisado (self-review no diff) e linkado à issue
- [ ] Swagger atualizado (quando houver rota nova)