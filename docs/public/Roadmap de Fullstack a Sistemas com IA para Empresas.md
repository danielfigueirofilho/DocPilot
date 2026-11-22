# Roadmap: de Fullstack a Sistemas com IA para Empresas

Nov 22, 2026 · @Daniel

## Como usar este roadmap

São 13 milestones construídos em cima de um único projeto. Com 10 a 15 horas por semana, isso dá de 9 a 12 meses. Cada milestone termina com algo funcionando no projeto, não com um curso concluído.

**Estrutura de cada milestone**

- **Entregável (E)**: checkbox principal. Só marque quando todas as tasks abaixo dele estiverem feitas.
- **Tasks**: checkboxes recuados sob cada entregável.
- **Estudar**: conceitos para ler antes ou durante.
- **Pronto quando**: o teste objetivo que fecha o milestone.

**Regras**

1. A primeira implementação de cada conceito é feita à mão. A IA pode explicar, revisar e gerar testes, mas não escreve o código da primeira vez.
2. Cada milestone fecha com uma tag no git (v0.1, v0.2…), um README da feature e um ADR curto: a decisão tomada e a alternativa descartada.
3. Escreva uma nota pública do que aprendeu por milestone. Isso vira portfólio e material de venda.
4. Travou mais de uma semana num item? Anote o bloqueio, siga em frente e volte depois.
5. M0 a M8 seguem em ordem. M9 a M12 podem se sobrepor. A trilha de negócio roda em paralelo desde o primeiro mês.

## O projeto-âncora: DocPilot (nome provisório)

O DocPilot é um SaaS B2B multi-tenant que lê documentos de empresas, extrai dados, responde perguntas com citação e envia o resultado para o ERP ou CRM do cliente. Ele foi escolhido porque obriga a passar por todos os tópicos que uma empresa grande cobra.

**O que ele faz**

- A empresa sobe contratos, propostas ou notas fiscais em PDF.
- Um pipeline em fila extrai o texto, divide em trechos e gera embeddings.
- A IA extrai campos estruturados (partes, valores, prazos) e valida regras de negócio.
- Usuários fazem perguntas no chat e recebem respostas com link para documento e página, vendo só o que têm permissão.
- Um agente lança os dados aprovados no sistema do cliente via API ou webhook.
- Cobrança por assento mais créditos de processamento, com SSO e trilha de auditoria.

O tipo de documento é um placeholder. A trilha de negócio define o nicho real (contabilidade, jurídico, compras), e você ajusta os campos extraídos sem mudar a arquitetura.

**Stack alvo**

| Camada | Escolha | Observação |
| --- | --- | --- |
| Monorepo | pnpm workspaces + Turborepo | apps/web, apps/api, apps/worker, packages/db, packages/shared |
| Front | Next.js + TypeScript | O que você já usa |
| API | Express + TypeScript | Separada do Next para ter camadas claras |
| Worker | Node + BullMQ | Processo próprio, escala separado da API |
| Banco | PostgreSQL + pgvector | Postgres puro em Docker, sem Supabase, para aprender RLS na mão |
| ORM | Drizzle | Mais simples para `SET LOCAL` de tenant em transação |
| Fila e cache | Redis | Local em Docker, ElastiCache no M11 |
| Arquivos | S3 | MinIO local |
| IA | Vercel AI SDK, Mastra no agente | Depois de fazer tudo à mão no M5 |
| Auth | Better Auth | Plugins de organização e SSO |
| Observabilidade | Pino, Sentry, OpenTelemetry, Langfuse | Entra no M0 e cresce |
| Infra | Docker, GitHub Actions, Terraform, AWS | AWS só no M11 |
| Cobrança | Stripe ou Asaas | Decisão no M10 |

&#91;embedded content: arquitetura alvo · 8 componentes\]

A API nunca faz trabalho pesado: ela grava no banco e enfileira. O worker lê o arquivo do S3, chama o LLM e grava o resultado. A arquitetura completa só fica pronta no M8, mas ela já é o alvo desde o M0.

## Visão geral

&#91;embedded content: roadmap · 4 fases, 13 milestones, 3 gates\]

Cada gate é a condição para mudar de fase: só avance quando os critérios "Pronto quando" da fase anterior estiverem cumpridos.

## M0 · Fundação do projeto (1 a 2 semanas)

O projeto já nasce com CI, logs e rastreio de erros, e tudo o que vem depois usa isso.

- [ ] **E0.1 Monorepo configurado**
  - [ ] pnpm workspaces + Turborepo com apps/web, apps/api, apps/worker, packages/db, packages/shared
  - [ ] tsconfig base com `strict` e `noUncheckedIndexedAccess`
  - [ ] ESLint + Prettier rodando em todos os pacotes
  - [ ] Variáveis de ambiente validadas com Zod: a app não sobe se faltar alguma
- [ ] **E0.2 Ambiente local com Docker**
  - [ ] docker-compose com Postgres + pgvector, Redis e MinIO
  - [ ] `pnpm dev` sobe tudo com um comando
  - [ ] `pnpm db:reset` recria o banco e roda o seed
- [ ] **E0.3 Observabilidade mínima**
  - [ ] Pino na API com logs em JSON
  - [ ] Middleware de requestId: gera ou propaga `X-Request-Id` e grava em todo log
  - [ ] Sentry na API e no Next, com ambiente e versão do release
  - [ ] Rotas `/health` (processo vivo) e `/ready` (banco e Redis respondendo)
- [ ] **E0.4 CI no GitHub Actions**
  - [ ] Workflow em todo PR: install com cache, lint, `tsc --noEmit`, testes
  - [ ] Postgres como service container no job de testes
  - [ ] Proteção da branch main: merge só com CI verde

**Estudar**

- [ ] GitHub Actions: jobs, services, cache e secrets
- [ ] Conventional Commits e PRs pequenos
- [ ] 12-Factor App, principalmente config e logs

**Pronto quando:** um PR com teste quebrado é bloqueado, e um erro forçado na API aparece no Sentry com o mesmo requestId do log.

## M1 · Arquitetura em camadas e erros (2 a 3 semanas)

A regra de negócio fica separada do HTTP e do banco, e todo erro tem formato previsível. Entidades iniciais: Organization, User, Membership, Document, Extraction.

- [ ] **E1.1 Camadas separadas**
  - [ ] Estrutura routes/controllers → services (casos de uso) → repositories
  - [ ] Controller só valida com Zod e traduz HTTP; service não importa Express nem Drizzle
  - [ ] Injeção de dependência simples por construtor ou factory, sem framework
  - [ ] CRUD de Document passando pelas três camadas
- [ ] **E1.2 Erros estruturados**
  - [ ] Classe `AppError` com `code`, `httpStatus` e `isOperational`
  - [ ] Subclasses: NotFoundError, ValidationError, ForbiddenError, ConflictError
  - [ ] Global error handler: loga stack e contexto, devolve `{ code, message, requestId }` sem vazar detalhes internos
  - [ ] Formato de erro documentado, usando a RFC 9457 (Problem Details) como referência
  - [ ] `unhandledRejection` e `uncaughtException` logados, com graceful shutdown
- [ ] **E1.3 Testes por camada**
  - [ ] Unitários dos services com repositório fake em memória
  - [ ] Integração das rotas com Supertest e banco real de teste
  - [ ] Testes dos caminhos de erro, não só do caminho feliz
- [ ] **E1.4 Contrato da API**
  - [ ] OpenAPI gerado a partir dos schemas Zod
  - [ ] Rotas versionadas em `/v1`

**Estudar**

- [ ] Do SOLID, só dois: Responsabilidade Única e Inversão de Dependência
- [ ] Arquitetura hexagonal em versão leve (ports and adapters)
- [ ] Pirâmide de testes
- [ ] RFC 9457 (Problem Details)

**Pronto quando:** trocar a implementação de um repository não exige mudar nenhum service nem teste unitário.

## M2 · PostgreSQL a fundo (3 a 4 semanas)

Os dados ficam corretos sob concorrência e as queries continuam rápidas com volume real.

- [ ] **E2.1 Modelagem completa**
  - [ ] Tabelas: tenants, users, memberships, documents, document\_chunks, extractions, usage\_events
  - [ ] Constraints no banco: FK, UNIQUE, CHECK e NOT NULL, com o banco como última linha de defesa
  - [ ] Tipo da chave primária (UUID v7 ou bigint) decidido num ADR
  - [ ] Datas sempre em `timestamptz`
- [ ] **E2.2 Índices e performance**
  - [ ] Seed com 1 milhão de documentos usando `generate_series`
  - [ ] `EXPLAIN (ANALYZE, BUFFERS)` das 5 queries principais, antes e depois dos índices
  - [ ] Índice B-tree composto `(tenant_id, created_at)`
  - [ ] Índice parcial (`WHERE deleted_at IS NULL`) e GIN para JSONB
  - [ ] Paginação por cursor (keyset) no lugar de OFFSET
  - [ ] `pg_stat_statements` ativo para achar queries lentas
- [ ] **E2.3 Transações e concorrência**
  - [ ] Feature de créditos: cada processamento consome créditos do tenant
  - [ ] Teste que dispara 50 requisições simultâneas e reproduz a race condition
  - [ ] Correção com `SELECT … FOR UPDATE`
  - [ ] Segunda correção com UPDATE atômico condicional (`WHERE saldo >= custo`), comparando as duas no ADR
  - [ ] Controle otimista com coluna `version` na edição de documento, retornando 409
  - [ ] Retry automático em erro de serialização
- [ ] **E2.4 Migrations seguras**
  - [ ] Renomear uma coluna sem downtime usando expand/contract
  - [ ] Criar índice com `CREATE INDEX CONCURRENTLY`
  - [ ] Migrations rodando no CI contra um banco limpo

**Estudar**

- [ ] Use The Index, Luke (site gratuito sobre índices)
- [ ] MVCC e como o Postgres lida com versões de linha
- [ ] Locks e níveis de isolamento (Read Committed × Serializable)
- [ ] Connection pooling com PgBouncer

**Pronto quando:** o teste de 50 requisições simultâneas nunca deixa saldo negativo, e a listagem com 1 milhão de linhas usa índice no plano de execução (sem Seq Scan).

## M3 · Multi-tenancy, autenticação e auditoria (3 a 4 semanas)

O banco impede vazamento entre empresas mesmo se o código errar, e toda ação crítica fica registrada.

- [ ] **E3.1 Autenticação com organizações**
  - [ ] Better Auth com plugin de organização: criar org, convidar por e-mail, aceitar convite
  - [ ] Sessão segura: cookie httpOnly, SameSite, rotação e logout em todos os dispositivos
  - [ ] 2FA por TOTP, opcional por usuário
- [ ] **E3.2 Autorização**
  - [ ] RBAC com papéis owner, admin, member e viewer
  - [ ] Matriz de permissões definida em um único lugar do código
  - [ ] ABAC simples: documento pertence a um departamento, e o membro só vê departamentos liberados
  - [ ] Checagem de permissão no service (não só no front), com teste para cada papel
- [ ] **E3.3 Isolamento no banco com RLS**
  - [ ] RLS habilitado em todas as tabelas com `tenant_id`, com `FORCE ROW LEVEL SECURITY`
  - [ ] Usuário de banco da aplicação sem `BYPASSRLS` e sem ser dono das tabelas
  - [ ] Tenant definido por transação com `SET LOCAL app.tenant_id` em cada request
  - [ ] Suíte cross-tenant: tenant A tenta ler, editar e apagar dados de B por todas as rotas
- [ ] **E3.4 Audit log**
  - [ ] Tabela `audit_events` só de inserção (o usuário da app não tem UPDATE nem DELETE)
  - [ ] Registro de quem, o quê, quando, IP e antes/depois em ações críticas: login, troca de papel, exclusão, export
  - [ ] Tela de auditoria para admin, com filtros e export CSV

**Estudar**

- [ ] Autenticação × autorização, e sessão × JWT
- [ ] OWASP ASVS: capítulos de autenticação e sessão
- [ ] Row-Level Security no Postgres
- [ ] Modelos de multi-tenancy: banco compartilhado, schema por cliente, banco por cliente

**Pronto quando:** remover de propósito um filtro `tenant_id` do código não vaza nenhum dado, porque o RLS bloqueia e a suíte cross-tenant pega.

## M4 · Filas e pipeline de ingestão (3 a 4 semanas)

O upload de um documento é processado em segundo plano, com retry, sem travar a API e sem duplicar nada.

- [ ] **E4.1 Upload seguro**
  - [ ] Upload direto para S3/MinIO por URL pré-assinada (o arquivo não passa pela API)
  - [ ] Validação do tipo real do arquivo (magic bytes) e tamanho máximo
  - [ ] Status do documento: uploaded → processing → ready ou failed
- [ ] **E4.2 Pipeline com BullMQ**
  - [ ] Etapas encadeadas: extrair texto → dividir em trechos → gerar embeddings → indexar
  - [ ] Extração de texto de PDF e OCR para PDF escaneado (Tesseract ou AWS Textract)
  - [ ] Retry com backoff exponencial e limite de tentativas
  - [ ] Fila de falhas (dead-letter) com tela para reprocessar
  - [ ] Limite de concorrência por tenant, para um cliente grande não travar os outros
- [ ] **E4.3 Idempotência e consistência**
  - [ ] Header `Idempotency-Key` no endpoint de criação: mesma chave, mesma resposta
  - [ ] Jobs idempotentes: reprocessar um documento não duplica trechos (upsert por hash)
  - [ ] Outbox pattern: o evento é gravado na mesma transação do registro e publicado por um relay
- [ ] **E4.4 Status e operação**
  - [ ] Progresso enviado ao front por SSE (ou polling simples)
  - [ ] Bull Board protegido por login para inspecionar filas
  - [ ] Graceful shutdown do worker: termina o job atual antes de sair

**Estudar**

- [ ] Entrega at-least-once × exactly-once
- [ ] Idempotência e outbox pattern
- [ ] Backpressure e rate limiting em filas
- [ ] Redis básico: estruturas de dados e persistência

**Pronto quando:** matar o worker no meio de 100 uploads e subir de novo termina com 100 documentos prontos e nenhum trecho duplicado.

## M5 · Fundamentos de IA sem caixa-preta (3 a 4 semanas)

Você entende o que cada camada do SDK faz porque já fez tudo à mão antes. Esse é o milestone que resolve o que ficou terceirizado na LangChain.

- [ ] **E5.1 IA à mão, sem SDK**
  - [ ] Chamada à API com `fetch`: mensagens, system prompt, temperature e max tokens
  - [ ] Streaming lendo o SSE do provedor e repassando ao front
  - [ ] Loop de tool calling escrito à mão: o modelo pede a tool, você executa, devolve o resultado e repete até a resposta final, com limite de passos
- [ ] **E5.2 Extração estruturada**
  - [ ] Schema Zod dos campos (ex.: contrato com partes, valor, vigência, multa e foro)
  - [ ] Structured output com Vercel AI SDK e validação do retorno contra o schema
  - [ ] O que dá para extrair por regra (XML de NF-e, regex de CNPJ) não vai para o LLM
  - [ ] Validações depois da extração: CNPJ válido, datas coerentes, totais que batem
- [ ] **E5.3 Custo, latência e falha**
  - [ ] Tokens de entrada e saída contados, com custo gravado por chamada e por tenant em `usage_events`
  - [ ] Timeout, retry com backoff em erro 429 e 5xx, e fallback para outro modelo ou provedor
  - [ ] Cache por hash do documento + versão do prompt
  - [ ] Prompts versionados no repositório, tratados como código
- [ ] **E5.4 Embeddings**
  - [ ] Embeddings gerados em lote no worker
  - [ ] Entender dimensão, similaridade de cosseno e por que trocar de modelo exige reindexar tudo

**Estudar**

- [ ] Tokenização e janela de contexto
- [ ] Temperature e top-p
- [ ] Tool calling e structured outputs na documentação do provedor que você usa
- [ ] Tabela de preços dos modelos e como estimar custo por documento

**Pronto quando:** você desenha o loop de tool calling sem consultar nada e sabe quanto custou cada documento processado.

## M6 · Evals e observabilidade de IA (2 a 3 semanas)

Medir vem antes de otimizar: toda mudança de prompt, modelo ou chunking vira um número comparado com o anterior.

- [ ] **E6.1 Dataset de avaliação**
  - [ ] 30 a 50 documentos realistas e anonimizados, com gabarito dos campos
  - [ ] 30 perguntas com resposta esperada e documento-fonte, para o RAG do M7
  - [ ] Casos difíceis de propósito: escaneado ruim, campo ausente, formato fora do padrão
- [ ] **E6.2 Runner de evals**
  - [ ] Métricas determinísticas: acerto por campo, exact match e F1
  - [ ] LLM-as-judge para respostas abertas, com rubrica escrita e revisão manual de uma amostra
  - [ ] Relatório comparando a versão atual com a anterior
- [ ] **E6.3 Evals no CI**
  - [ ] Subset rápido rodando em PRs que mexem em prompts ou código de IA
  - [ ] Baseline salvo no repositório; o PR é bloqueado se cair abaixo dele
- [ ] **E6.4 Tracing de IA**
  - [ ] Langfuse com um trace por request: prompt, resposta, tokens, custo, latência e tools chamadas
  - [ ] traceId do Langfuse ligado ao requestId do Pino
  - [ ] Feedback positivo/negativo do usuário gravado e ligado ao trace

**Estudar**

- [ ] Precisão, recall e F1
- [ ] Vieses do LLM-as-judge e como calibrar com revisão humana
- [ ] Overfitting ao dataset de eval

**Pronto quando:** você troca o modelo de extração e, em minutos, sabe se a precisão subiu ou caiu em cada campo.

## M7 · RAG de produção (3 a 4 semanas)

As respostas citam a fonte, respeitam permissões e têm qualidade medida pelo eval do M6.

- [ ] **E7.1 Chunking**
  - [ ] Três estratégias comparadas no eval: tamanho fixo com overlap, por estrutura (títulos e seções) e semântica
  - [ ] Metadados em cada trecho: tenant, documento, página, seção e departamento
- [ ] **E7.2 Busca**
  - [ ] Índice HNSW no pgvector, entendendo `m`, `ef_construction` e `ef_search`
  - [ ] Busca por palavra-chave com `tsvector` em português e índice GIN
  - [ ] Hybrid search juntando as duas com Reciprocal Rank Fusion
  - [ ] Reranking dos melhores resultados antes de mandar ao LLM
  - [ ] recall@k e MRR medidos antes e depois de cada etapa
- [ ] **E7.3 Permissões no RAG**
  - [ ] Filtro de tenant e departamento aplicado dentro da query vetorial (RLS + WHERE), nunca depois
  - [ ] Teste: usuário sem acesso pergunta sobre um documento restrito e não recebe nenhum trecho dele
- [ ] **E7.4 Resposta**
  - [ ] Citação obrigatória com link para documento e página
  - [ ] Resposta "não encontrei" quando o contexto recuperado não é suficiente
  - [ ] Streaming da resposta no chat

**Estudar**

- [ ] Busca aproximada (ANN) × busca exata
- [ ] BM25 × full-text do Postgres
- [ ] Reciprocal Rank Fusion e rerankers
- [ ] Reescrita de pergunta (query rewriting)

**Pronto quando:** o eval mostra ganho medido do hybrid + rerank sobre a busca só vetorial, e o teste de permissão passa. A partir daqui o projeto já serve de demo para a trilha de negócio.

## M8 · Agente, guardrails e integrações (4 a 5 semanas)

A IA age em sistemas de terceiros com segurança, e toda ação de escrita passa por aprovação humana.

- [ ] **E8.1 Agente**
  - [ ] Agente (Mastra ou AI SDK) com tools: buscar documentos, consultar extrações, criar registro no sistema externo
  - [ ] Limite de passos, timeout total e orçamento de tokens por execução
  - [ ] Human-in-the-loop: ações de escrita ficam pendentes até um humano aprovar
- [ ] **E8.2 Guardrails**
  - [ ] Casos de prompt injection indireta no eval: documento com instrução escondida no texto
  - [ ] Dados separados de instruções no prompt
  - [ ] Tools executam com a permissão de quem pediu, nunca com permissão de admin
  - [ ] Saída validada por schema e bloqueio de vazamento do system prompt
  - [ ] Recusa de temas fora do escopo do cliente
  - [ ] Red team com 20 ataques no dataset de eval, rodando no CI
- [ ] **E8.3 Integrações de saída**
  - [ ] Webhooks para o cliente com assinatura HMAC, timestamp, retries, log de entregas e reenvio manual
  - [ ] Uma integração real com ERP ou CRM que tenha API e ambiente de teste (ex.: Omie, Bling, Pipedrive, HubSpot)
  - [ ] Import e export CSV para clientes sem API
- [ ] **E8.4 API pública**
  - [ ] API keys por tenant: guardadas com hash, com escopo, rotação e revogação
  - [ ] Rate limiting por chave com Redis (token bucket)
  - [ ] Documentação OpenAPI pública com exemplos

**Estudar**

- [ ] OWASP Top 10 for LLM Applications
- [ ] Padrões de agentes: ReAct e planner/executor
- [ ] Boas práticas de design de webhooks
- [ ] OAuth2 client credentials (integração entre sistemas)

**Pronto quando:** o agente lança uma extração aprovada no sistema externo, e todos os ataques do red team são bloqueados ou ficam presos na aprovação humana.

## M9 · Segurança de aplicação e LGPD (3 a 4 semanas)

Você consegue provar, com evidência, como o sistema protege os dados. É isso que o time de segurança do cliente vai perguntar.

- [ ] **E9.1 OWASP Top 10**
  - [ ] Checklist do OWASP Top 10 aplicado ao projeto, com evidência em cada item
  - [ ] Headers de segurança (helmet, CSP), CORS restrito e proteção CSRF onde há cookie
  - [ ] Rate limiting em login e em endpoints caros, com bloqueio por tentativas
  - [ ] Proteção contra SSRF em qualquer lugar que busque uma URL
- [ ] **E9.2 Dependências e segredos**
  - [ ] Dependabot e `pnpm audit` no CI; CodeQL para análise estática
  - [ ] gitleaks no CI e processo de rotação de segredos documentado
  - [ ] Imagem Docker mínima, rodando sem root, escaneada com Trivy
- [ ] **E9.3 Dados pessoais**
  - [ ] Inventário de dados pessoais: qual dado, onde fica, para quê e por quanto tempo
  - [ ] Mascaramento de dados pessoais (CPF, e-mail, telefone) antes de enviar ao LLM, quando o cliente exigir
  - [ ] Criptografia de campos sensíveis com envelope encryption (a chave vai para o KMS no M11)
  - [ ] Retenção configurável por tenant
  - [ ] Exclusão completa, incluindo arquivos, trechos e embeddings
  - [ ] Export dos dados de um titular
- [ ] **E9.4 Documentos de segurança**
  - [ ] Threat model leve (STRIDE) do fluxo de documentos
  - [ ] Política de privacidade, termos de uso e minuta de DPA, revisados por advogado
  - [ ] Lista de subprocessadores (provedor de IA, cloud, e-mail)

**Estudar**

- [ ] LGPD: bases legais, controlador × operador, direitos do titular, comunicação de incidente à ANPD
- [ ] OWASP ASVS nível 1
- [ ] Noções de SOC 2 e ISO 27001 (o que são e o que pedem)

**Pronto quando:** você responde um questionário de segurança padrão sem inventar nada, com evidência para cada resposta.

## M10 · Billing e produto SaaS (3 a 4 semanas)

O cliente assina, usa, paga e cancela sozinho, e você sabe a margem de cada um depois do custo de IA.

- [ ] **E10.1 Planos e cobrança**
  - [ ] Modelo de preço: assinatura por assento + créditos de processamento
  - [ ] Gateway escolhido num ADR: Stripe (Checkout, Billing, Customer Portal) ou Asaas/Mercado Pago se o público pagar com Pix e boleto
  - [ ] Checkout e portal do cliente integrados
- [ ] **E10.2 Webhooks de pagamento**
  - [ ] Verificação de assinatura e processamento via fila
  - [ ] Handler idempotente: o id do evento é gravado e eventos repetidos são ignorados
  - [ ] Estados da assinatura (trial, active, past\_due, canceled) e o que o usuário pode fazer em cada um
  - [ ] Cobrança falha: aviso, período de carência e bloqueio
- [ ] **E10.3 Medição e limites**
  - [ ] Eventos de uso do M5 agregados por tenant e período
  - [ ] Limites do plano aplicados no backend: documentos por mês, assentos e créditos
  - [ ] Painel de consumo para o cliente e alerta ao atingir 80% do limite
  - [ ] Relatório interno de margem: receita × custo de IA por tenant
- [ ] **E10.4 Onboarding**
  - [ ] Primeiro uso guiado: criar org → subir documento de exemplo → ver a extração
  - [ ] E-mails transacionais via fila: convite, documento pronto, falha, cobrança
  - [ ] Emissão de nota fiscal de serviço resolvida (ferramenta ou contador)

**Estudar**

- [ ] Métricas SaaS: MRR, churn, CAC e LTV (sua faculdade de Administração ajuda aqui)
- [ ] Precificação por valor entregue × por custo
- [ ] Como os gateways tratam reembolso, chargeback e disputa

**Pronto quando:** um cliente de teste assina, estoura o limite, faz upgrade e cancela sem você mexer em nada manualmente.

## M11 · Cloud AWS e operação (4 a 6 semanas)

O sistema roda numa infraestrutura que o time de TI do cliente reconhece, e você sabe se recuperar de uma falha.

- [ ] **E11.1 Infra como código**
  - [ ] Terraform com VPC e subnets privadas, RDS Postgres, ElastiCache Redis, S3 e ECS Fargate (API e worker)
  - [ ] Front no Vercel ou no ECS, decidido num ADR
  - [ ] Ambientes de staging e produção separados
  - [ ] Região sa-east-1 (São Paulo) para manter os dados no Brasil
  - [ ] Alerta de orçamento na conta; staging destruído quando não estiver em uso
- [ ] **E11.2 Segurança da conta**
  - [ ] IAM com menor privilégio, MFA e nenhum uso da conta root
  - [ ] Segredos no Secrets Manager ou SSM; chaves de criptografia do M9 no KMS
  - [ ] Banco e Redis sem acesso público
- [ ] **E11.3 Deploy**
  - [ ] GitHub Actions → build da imagem → ECR → deploy no ECS, autenticando por OIDC (sem chave AWS fixa)
  - [ ] Migrations como etapa do deploy e rollback documentado
- [ ] **E11.4 Operação**
  - [ ] OpenTelemetry com um trace só cobrindo API → fila → worker → LLM
  - [ ] Alertas: taxa de erro, latência p95, fila parada e custo de IA fora do normal
  - [ ] Backups automáticos e um teste real de restore, com RPO e RTO definidos
  - [ ] Teste de carga com k6 e os limites encontrados registrados
  - [ ] Runbook de incidentes e status page

**Estudar**

- [ ] Conteúdo do AWS Cloud Practitioner e depois do Solutions Architect Associate
- [ ] Redes: VPC, subnet, NAT, security group
- [ ] SLI, SLO e SLA

**Pronto quando:** você recria o staging do zero com um comando, restaura um backup e sabe quanto a infraestrutura custa por mês.

## M12 · Enterprise readiness e piloto (4 a 6 semanas, depois contínuo)

O projeto vira produto vendável: SSO, contrato de IA adequado, pacote de segurança e um piloto pago.

- [ ] **E12.1 SSO e provisionamento**
  - [ ] SSO via SAML ou OIDC com Microsoft Entra ID e Google Workspace (plugin SSO do Better Auth ou serviço como WorkOS)
  - [ ] Domínio da empresa verificado, com login obrigatório por SSO
  - [ ] SCIM para criar e remover usuários automaticamente (opcional, mas pedido por empresas grandes)
- [ ] **E12.2 IA com contrato enterprise**
  - [ ] Provedor com contrato que garanta não usar os dados para treino e retenção controlada (plano enterprise da API, Azure OpenAI ou AWS Bedrock)
  - [ ] Provedor e região do modelo configuráveis por tenant
- [ ] **E12.3 Pacote de venda**
  - [ ] Documento de arquitetura e segurança: diagrama, fluxo de dados, onde os dados ficam, criptografia, retenção e subprocessadores
  - [ ] Respostas prontas para questionário de segurança
  - [ ] Minuta de SLA: disponibilidade, tempo de resposta a incidente e canais de suporte
  - [ ] Contrato de prestação de serviço revisado por advogado
- [ ] **E12.4 Piloto**
  - [ ] 1 a 3 pilotos com empresas de médio porte vindas da trilha de negócio
  - [ ] Métricas de sucesso combinadas antes: horas economizadas, taxa de acerto, uso semanal
  - [ ] Estudo de caso escrito no final de cada piloto

**Estudar**

- [ ] SAML × OIDC e como funciona o fluxo de SSO
- [ ] SCIM
- [ ] Termos de uso de dados dos provedores de LLM que você usa

**Pronto quando:** um piloto vira contrato pago.

## Trilha paralela de negócio

Essa trilha roda junto com a técnica desde o primeiro mês. Sem ela, você termina o M12 com um sistema ótimo e nenhum cliente esperando por ele.

- [ ] **N1 Descoberta (meses 1 a 3)**
  - [ ] Escolher 2 ou 3 nichos candidatos (ex.: escritórios de contabilidade, jurídico de médias empresas, compras e suprimentos)
  - [ ] 10 entrevistas sobre o problema, não sobre o produto: como fazem hoje, quanto custa, quem decide a compra
  - [ ] Escolher 1 nicho e 1 dor, e ajustar os campos extraídos do DocPilot a ela
- [ ] **N2 Estrutura (meses 3 a 6)**
  - [ ] Definir o sócio que cuida de vendas e contrato
  - [ ] Definir a estrutura societária e validar com o jurídico as regras da sua situação profissional
  - [ ] Landing page com lista de espera e um vídeo curto de demo
  - [ ] Um post técnico publicado por milestone (LinkedIn ou blog)
- [ ] **N3 Validação (meses 6 a 9, depois do M7)**
  - [ ] 5 empresas testando o MVP sem custo em troca de feedback semanal
  - [ ] Preço definido a partir do valor percebido e do custo de IA medido
  - [ ] Proposta comercial padrão pronta
- [ ] **N4 Venda (meses 9 a 12)**
  - [ ] Pilotos do M12 rodando
  - [ ] Primeiro contrato pago
  - [ ] Processo de suporte definido: canal, horário e prazo de resposta

**Estudar**

- [ ] Livro *The Mom Test* (como entrevistar clientes sem enviesar)
- [ ] Venda B2B consultiva e ciclo de compra em empresas
- [ ] Como montar uma proposta comercial

## Checklist final: pronto para vender para empresas

Se todos os itens abaixo estiverem marcados, você tem o que um time de TI e segurança costuma exigir antes de aprovar um fornecedor.

- [ ] Isolamento entre clientes garantido pelo banco (RLS) e coberto por testes
- [ ] Papéis e permissões testados, inclusive dentro do RAG
- [ ] Audit log imutável e exportável
- [ ] SSO com Entra ID e Google Workspace
- [ ] Filas com retry, fila de falhas e idempotência
- [ ] Evals com baseline rodando no CI
- [ ] Custo de IA conhecido por cliente e por documento
- [ ] Logs, traces e alertas que avisam antes do cliente
- [ ] Backup restaurado com sucesso pelo menos uma vez
- [ ] Dados no Brasil e provedor de IA com contrato de não-treino
- [ ] Inventário LGPD, política de privacidade e DPA
- [ ] Documento de arquitetura e segurança e questionário respondido
- [ ] Contrato e SLA revisados por advogado
- [ ] Sócio comercial definido e primeiro cliente pagante
