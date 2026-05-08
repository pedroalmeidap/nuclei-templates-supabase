# Nuclei Templates — Supabase RLS Misconfiguration

Templates para detectar misconfigurações de Row Level Security (RLS) em instâncias Supabase expostas via anon key.

## Contexto

A anon key do Supabase é **pública por design** — ela fica embutida nos bundles JS do frontend. O problema não é a chave em si, mas o que ela consegue acessar.

Quando tabelas são criadas sem RLS habilitado, ou com políticas `USING (true)`, qualquer pessoa com a anon key consegue ler todos os registros via REST API (`/rest/v1/<tabela>`).

## Templates

| Template | Severidade | O que detecta |
|---|---|---|
| `supabase-anon-key-data-exposure.yaml` | High | Tabelas que retornam qualquer dado via anon key |
| `supabase-sensitive-fields-exposure.yaml` | Critical | Campos sensíveis expostos (Stripe, CPF, planos, comissões) |
| `supabase-rls-disabled-check.yaml` | High | Tabelas sem RLS habilitado (retornam contagem via Content-Range) |

## Pré-requisitos

```bash
# Instalar Nuclei
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest

# Ou via brew
brew install nuclei
```

## Como obter a anon key

A anon key fica nos bundles JS do frontend. Para encontrá-la:

1. Abra o site no browser com Burp Suite ou DevTools (Network)
2. Navegue para uma página que usa autenticação (ex: redefinição de senha, login)
3. Filtre por `supabase.co` nas requisições
4. A anon key aparece nos headers `apikey` e `Authorization`

Alternativamente, busque nos chunks JS por `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9` (prefixo padrão de JWT Supabase).

## Uso

### Rodar todos os templates

```bash
nuclei -t . \
  -var anon_key="eyJhbGci..." \
  -var supabase_url="https://SEU-ID.supabase.co" \
  -o resultados.txt
```

### Rodar template específico

```bash
# Buscar qualquer dado exposto
nuclei -t supabase-anon-key-data-exposure.yaml \
  -var anon_key="eyJhbGci..." \
  -var supabase_url="https://SEU-ID.supabase.co"

# Focar em campos sensíveis (Stripe, PII, planos)
nuclei -t supabase-sensitive-fields-exposure.yaml \
  -var anon_key="eyJhbGci..." \
  -var supabase_url="https://SEU-ID.supabase.co"

# Detectar tabelas sem RLS
nuclei -t supabase-rls-disabled-check.yaml \
  -var anon_key="eyJhbGci..." \
  -var supabase_url="https://SEU-ID.supabase.co"
```

### Usar wordlist customizada

```bash
nuclei -t supabase-anon-key-data-exposure.yaml \
  -var anon_key="eyJhbGci..." \
  -var supabase_url="https://SEU-ID.supabase.co" \
  -var table=minha_tabela
```

## Wordlist

`wordlists/saas-tables-pt.txt` — ~200 nomes de tabelas comuns em SaaS/BaaS brasileiro, cobrindo:

- Multi-tenant (tenants, empresas, workspaces)
- Pagamento (stripe_customers, subscriptions, invoices)
- CRM (clients, clientes, leads, contacts)
- Agendamento (appointments, bookings, availability)
- Saúde/Healthtech (anamnesis, patients, prontuarios)
- Educação/EAD (courses, enrollments, students)
- Marketing (campaigns, content_items, funnels)
- Afiliados/Indicações (affiliates, mentor_profiles, commissions)

## Verificação manual

Após encontrar uma tabela exposta, valide manualmente:

```bash
# Contar registros
curl 'https://SEU-ID.supabase.co/rest/v1/TABELA?select=count' \
  -H "apikey: SUA_ANON_KEY" \
  -H "Authorization: Bearer SUA_ANON_KEY" \
  -H "Prefer: count=exact"
# Checar header Content-Range: */TOTAL

# Ler dados (limit seguro para evidência)
curl 'https://SEU-ID.supabase.co/rest/v1/TABELA?select=*&limit=1' \
  -H "apikey: SUA_ANON_KEY" \
  -H "Authorization: Bearer SUA_ANON_KEY"
```

## Correção

```sql
-- 1. Habilitar RLS na tabela
ALTER TABLE nome_da_tabela ENABLE ROW LEVEL SECURITY;

-- 2. Remover política permissiva
DROP POLICY IF EXISTS nome_da_policy ON nome_da_tabela;

-- 3. Auditar todas as tabelas sem RLS
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public' AND rowsecurity = false;

-- 4. Auditar políticas permissivas
SELECT tablename, policyname, qual
FROM pg_policies
WHERE schemaname = 'public' AND qual = 'true';
```

## Uso responsável

Estes templates são destinados a:
- Pentest com escopo autorizado
- Auditoria do próprio ambiente
- Pesquisa de segurança defensiva
- Bug bounty programs com Supabase no escopo

Não utilize contra sistemas sem autorização explícita.
