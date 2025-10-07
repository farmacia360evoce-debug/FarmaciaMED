# farmaciamed — Plataforma de consulta de ativos e geração de receitas

> MVP: área do prescritor (3 botões) + área da farmácia (catálogo de ativos) + autenticação.

## 🎯 Objetivo

Construir uma plataforma para **prescritores** consultarem ativos farmacêuticos, receberem **sugestões de formulações** (isolada, associada com justificativa e terapias em conjunto) e **gerarem receitas** para enviar ao paciente e à farmácia. A **farmácia** mantém o banco de dados de ativos (doses, dosagens, tipos de receita, posologia, formas farmacêuticas, recomendações e links científicos).

## 🔒 Acesso e Perfis

* **Prescritor** (CRM/CRMV/CRF/Outro): consulta ativos, gera receitas, salva presets.
* **Farmácia**: cadastra/edita ativos e parametrizações (formas, faixas de dose, links, alertas de controlados).
* **Admin (opcional no MVP)**: gestão do tenant e auditoria.

## 🧭 Fluxo do Usuário (alto nível)

1. **Login** → validação de perfil.
2. **Área do Prescritor** com 3 botões:

   * **Consulta por ativos** → busca → ficha do ativo (dose/dosagem, posologia, tipo de receita, formas, recomendações, links científicos) → **3 sugestões** (isolada / associada c/ justificativa / terapias em conjunto) → ao escolher uma sugestão → **Gerar Prescrição** → preencher dados do paciente → salvar/enviar.
   * **Gerador de receitas** → formulário livre (paciente + múltiplos ativos) com **faixas de dose sugeridas (mín–máx)** e campos de posologia/observações → **Enviar**.
   * **Cadastro de fórmulas favoritas** → criar/editar **presets** do prescritor → **Gerar receita** em 1 clique.
3. **Receita controlada**: se qualquer item for de **controle especial**, exibir **alerta** e o botão “**Emitir via Sistema Nacional/Estadual de Controle de Receituário**” (link configurável nas **Configurações da Farmácia**). Após emissão externa, a plataforma registra o número/identificador da receita.
4. **Envio**: toda receita gerada tem **botão de envio** para **Farmácia** (inbox/API/email) e para **Paciente** (email/SMS/WhatsApp*, via provedor configurável).

> *Integrações de envio são pluggáveis no MVP: começar com email; SMS/WhatsApp podem entrar como backlog.

## 🗂️ Estrutura de Telas (MVP)

* **Login/Recuperar senha**
* **Dashboard Prescritor**

  * Botão 1: **Consulta por ativos** → Lista → Ficha do ativo → Sugestões → Gerar Prescrição
  * Botão 2: **Gerador de receitas** (form livre)
  * Botão 3: **Minhas fórmulas favoritas (presets)**
  * **Minhas receitas** (histórico, status, reenvio)
* **Dashboard Farmácia**

  * **Ativos** (CRUD)
  * **Configurações** (formas farmacêuticas, provedores de envio, link do sistema de receituário controlado, políticas de dose)
  * **Receitas recebidas** (inbox)

## 📦 Modelagem de Dados (sugestão PostgreSQL)

```sql
-- Usuários e perfis
CREATE TYPE user_role AS ENUM ('PRESCRIBER','PHARMACY','ADMIN');
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  role user_role NOT NULL,
  full_name TEXT,
  document_id TEXT,          -- CRM/CRMV/etc
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Farmácias (multi-tenant opcional)
CREATE TABLE pharmacies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  cnpj TEXT,
  settings JSONB DEFAULT '{}'::jsonb,  -- links externos, provedores de envio, políticas
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Relacionamento usuário–farmácia (ex.: prescritores vinculados, equipe da farmácia)
CREATE TABLE user_pharmacies (
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  pharmacy_id UUID REFERENCES pharmacies(id) ON DELETE CASCADE,
  role_in_tenant TEXT DEFAULT 'MEMBER',
  PRIMARY KEY (user_id, pharmacy_id)
);

-- Ativos mantidos pela farmácia
CREATE TYPE receita_tipo AS ENUM ('SIMPLIFICADA','BRANCA','AMARELA','AZUL','B1','B2','C1','C2','C3','C4','C5','ESPECIAL');
CREATE TABLE ativos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pharmacy_id UUID REFERENCES pharmacies(id) ON DELETE CASCADE,
  nome TEXT NOT NULL,
  sinonimos TEXT[],
  tipo_receita receita_tipo,         -- quando aplicável
  dose_min NUMERIC,                  -- por unidade
  dose_max NUMERIC,
  unidade_dose TEXT,                 -- mg, g, %, UI etc.
  posologia_padrao TEXT,
  formas_farmaceuticas TEXT[],
  recomendacoes TEXT,
  interacoes TEXT,
  contraindicacoes TEXT,
  links_cientificos TEXT[],
  is_controlado BOOLEAN DEFAULT false,
  metadata JSONB DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Sugestões de formulação por ativo (opcional; pode ser gerado on-the-fly)
CREATE TYPE sugestao_tipo AS ENUM ('ISOLADA','ASSOCIADA','TERAPIA_CONJUNTO');
CREATE TABLE sugestoes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ativo_id UUID REFERENCES ativos(id) ON DELETE CASCADE,
  tipo sugestao_tipo NOT NULL,
  justificativa TEXT,
  composicao JSONB NOT NULL,     -- [{ativo_id|nome, dose, unidade, forma, qsp, observacoes}]
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Pacientes (mínimo para emissão/registro)
CREATE TABLE pacientes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  prescriber_id UUID REFERENCES users(id) ON DELETE SET NULL,
  full_name TEXT NOT NULL,
  document_id TEXT,             -- CPF ou outro
  birthdate DATE,
  contacts JSONB DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Receitas
CREATE TYPE receita_status AS ENUM ('DRAFT','SIGNED','SENT_TO_PHARMACY','SENT_TO_PATIENT');
CREATE TABLE receitas (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pharmacy_id UUID REFERENCES pharmacies(id) ON DELETE SET NULL,
  prescriber_id UUID REFERENCES users(id) ON DELETE SET NULL,
  patient_id UUID REFERENCES pacientes(id) ON DELETE SET NULL,
  itens JSONB NOT NULL,              -- lista de itens/ativos e doses
  posologia_geral TEXT,
  observacoes TEXT,
  is_controlada BOOLEAN DEFAULT false,
  externo_protocolo TEXT,            -- nr/URL/identificador do sistema de receituário
  pdf_url TEXT,                      -- arquivo gerado (futuro)
  status receita_status DEFAULT 'DRAFT',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Presets (fórmulas favoritas do prescritor)
CREATE TABLE presets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  prescriber_id UUID REFERENCES users(id) ON DELETE CASCADE,
  nome TEXT NOT NULL,
  payload JSONB NOT NULL,           -- itens, doses, forma, instruções
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Auditoria
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  action TEXT NOT NULL,
  entity TEXT NOT NULL,
  entity_id UUID,
  payload JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

## 🔗 API (rótulo inicial – OpenAPI 3.1 resumido)

```yaml
openapi: 3.1.0
info:
  title: farmaciamed API
  version: 0.1.0
security:
  - bearerAuth: []
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
paths:
  /auth/register:
    post:
      summary: Registro de usuário
  /auth/login:
    post:
      summary: Login e retorno de JWT
  /ativos:
    get:
      summary: Buscar ativos (query por nome/sinônimo)
    post:
      summary: (Farmácia) Criar ativo
  /ativos/{id}:
    get:
      summary: Detalhar ativo
    patch:
      summary: (Farmácia) Atualizar ativo
  /ativos/{id}/sugestoes:
    get:
      summary: Retorna 3 sugestões (isolada, associada c/ justificativa, terapias em conjunto)
  /presets:
    get:
      summary: Listar presets do prescritor
    post:
      summary: Criar/atualizar preset
  /receitas:
    post:
      summary: Gerar receita (de sugestão ou livre)
  /receitas/{id}/enviar:
    post:
      summary: Enviar receita para farmácia e/ou paciente
  /pacientes:
    post:
      summary: Criar/atualizar paciente
```

### Exemplo de payload de **Receita** (POST /receitas)

```json
{
  "pharmacy_id": "uuid",
  "patient": {"full_name": "Maria Silva", "document_id": "000.000.000-00"},
  "itens": [
    {
      "ativo_id": "uuid",
      "nome": "Melatonina",
      "dose": 3,
      "unidade": "mg",
      "forma": "cápsulas",
      "posologia": "1 cápsula ao deitar",
      "observacoes": "Uso noturno"
    }
  ],
  "posologia_geral": null,
  "observacoes": "Revisar em 30 dias",
  "is_controlada": false
}
```

## ✅ Regras de Negócio (MVP)

1. **Faixas de dose**: a ficha do ativo exibe **mín–máx** definidos pela farmácia; o prescritor pode ajustar **dentro** da faixa (ou fora, com justificativa – flaggable nas configs).
2. **Sugestões**: sempre exibir **3 variações** para cada ativo consultado. A **ASSOCIADA** deve trazer **justificativa** (racional científico). “Terapias em conjunto” podem conter múltiplos ativos.
3. **Controlados**: se qualquer item marcado `is_controlado=true` → mostrar **banner** + botão “Emitir via sistema de controle” (URL configurável por farmácia). Persistir `externo_protocolo` na receita.
4. **Envio**: no ato do envio, registrar destinos (farmácia, paciente) e status. Primeiro provedor: **email SMTP**.
5. **Auditoria**: logar ações sensíveis (login, CRUD de ativos, geração/alteração de receita, envio).

## 🧪 Critérios de Aceite (DoD)

* **Login**: JWT emitido, rotas protegidas; recuperação de senha por email.
* **Consulta por ativos**: busca por nome/sinônimo; ficha com campos completos; 3 sugestões visíveis; botão **Gerar Prescrição** preenche o form.
* **Gerador de receitas**: adicionar n ativos, ver faixas mín–máx, editar posologia/observações, validar campos obrigatórios, salvar.
* **Presets**: criar, listar, duplicar, excluir; gerar receita a partir do preset.
* **Controlados**: alerta + link externo funcional + captura do protocolo na receita.
* **Envio**: registro de envio e reenvio com status.
* **Farmácia/Ativos (CRUD)**: validar obrigatórios, permitir múltiplas formas, links científicos e flags de controlado.

## 🛡️ Segurança & LGPD

* Hash de senha com **argon2** (ou bcrypt com custo adequado).
* JWT curto + **refresh tokens**; **2FA** opcional (backlog).
* **Controle de acesso por papel** (RBAC) no backend.
* **Criptografia em repouso** para dados sensíveis do paciente (campos PII com pgcrypto/KMS).
* **Logs de auditoria** e **trilhas de consentimento** (aceite de termos pelo prescritor).
* Política de **retenção de dados** e **portabilidade** (download do prontuário/receitas do paciente quando aplicável).

## 🛠️ Stack Sugerida (opcional)

* **Frontend**: Next.js 15 (App Router), React 19, TypeScript, Tailwind, shadcn/ui.
* **Backend**: Node 22 + NestJS/Express; ou **FastAPI** em Python.
* **DB**: PostgreSQL + Prisma/TypeORM; **pgvector** (futuro para busca semântica).
* **Auth**: JWT + cookies httpOnly/secure.
* **Infra**: Docker + docker-compose; CI GitHub Actions; deploy Railway/Fly.io/Vercel.

## 🔧 Variáveis de Ambiente (exemplo)

```
DATABASE_URL=
JWT_SECRET=
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASS=
EXTERNAL_CONTROLLED_RX_URL=   # link do sistema de receituário controlado
```

## 📋 Backlog Inicial (Issues sugeridas)

* FE: Layout base + Auth pages.
* BE: Auth (register/login/refresh) + RBAC.
* BE: CRUD de ativos (Farmácia).
* FE/BE: Consulta por ativos + ficha + 3 sugestões.
* FE/BE: Gerador de receitas com faixas mín–máx.
* FE/BE: Presets (favoritos).
* BE: Envio por email (farmácia e paciente) + status de envio.
* BE: Auditoria (middleware).
* DevOps: Docker + CI + seed inicial.

## 🧱 Estrutura de Pastas (monorepo opcional)

```
/apps
  /web        # Next.js
  /api        # NestJS/FastAPI
/packages
  /ui         # componentes compartilhados
  /types      # tipos e contratos
/infra        # docker, compose, IaC
```

## 🖼️ UX de referência (rótulo rápido)

* **Dashboard Prescritor**: 3 cards grandes (Consulta | Gerador | Favoritas) + lista de “Minhas receitas”.
* **Ficha do Ativo**: header (nome + tipo de receita), tabs (Informações | Sugestões | Material científico), CTA “Gerar Prescrição”.
* **Gerador**: multi-steps (Paciente → Itens → Revisão → Envio).
* **Farmácia/Ativos**: tabela + drawer para edição rica (formas, links, flags, faixas de dose).

## 🧱 Validações de Dose (exemplo)

* **UI** mostra slider/inputs com **min–max**.
* Se o usuário extrapola a faixa e a farmácia permitir, exigir **justificativa** e registrar em `audit_logs`.

## 🔌 Integrações (MVP)

* **Email SMTP** para envio.
* **Link externo** configurável ao sistema de receituário controlado (estado/federal): `EXTERNAL_CONTROLLED_RX_URL`.

## 📝 Licença

MIT (sugerida) — ajustar conforme política da organização.

---

### Como começar (dev)

1. `docker compose up -d` (postgres + mailhog em dev).
2. `npm run db:migrate && npm run db:seed`.
3. `npm run dev` (web + api).

---

> Esta especificação é um **prompt/README inicial** para o repositório **farmaciamed**. A partir dele, crie as *issues* e itere no fluxo descrito acima. Ajustes de terminologia/regra podem ser feitos pela Farmácia nas **Configurações** do tenant.
