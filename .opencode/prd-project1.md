# PRD — Execução Adaptativa rumo ao Objetivo 3

## PENTEST-ROE-001 · Alfa Education (Alfa PTE)

| Campo | Valor |
|---|---|
| Documento | `prd-project1.md` |
| Projeto | Operação de pentest autorizada — Alfa PTE |
| Fonte de verdade do escopo | `project1.md` + `pte-Termo_Autorizacao_Pentest_ROE_001 (2) - Copia (1).docx` |
| Objetivo central | Objetivo 3 do RoE — download do banco de questões PTE existente no sistema |
| Versão | 1.0 |
| Data | 2026-11-08 |
| Status | Aprovado para execução (escopo confirmado pelo operador em 2026-11-08) |
| Janela RoE | 07/11/2026 a 18/12/2026 |
| Limite de taxa | Máximo 3 requisições por segundo (global) |
| Conta de teste | Permitida; pode ser criada pela equipe de red team |
| Idioma | Português |

---

## 1. Sumário executivo

Este PRD define **como** executar, de forma adaptativa e controlada, a fase de testes rumo ao
Objetivo 3 do RoE: obter o banco de questões do PTE existente no sistema da Alfa PTE sem danificar
o backend, sem afetar outros usuários e sem exceder os limites operacionais acordados.

A operação parte de uma conta de teste gratuita criada pela própria equipe, mapeia o modelo real
de entrega de questões (endpoints, filtros, paginação, permissões e arquivos de mídia), e evolui em
ciclos curtos de observação → hipótese → teste mínimo → decisão. Cada decisão é guiada por
evidência coletada, não por suposição. O plano prevê ramificações explícitas para os cenários mais
prováveis: acesso direto ao catálogo, catálogo mediado por mock tests/CYO, catálogo parcialmente
bloqueado por paywall, IDs previsíveis e entrega de mídia por URL pública ou assinada.

O sucesso é medido por cobertura comprovada do catálogo (questões, metadados e mídia associada),
integridade das evidências e ausência de impacto observável. Qualquer exposição de dados de
terceiros interrompe a coleta imediatamente, conforme a cláusula de tratamento de dados do RoE.

---

## 2. Contexto e fonte de verdade

### 2.1 Objetivo 3 no RoE

O RoE autoriza, na terceira linha da tabela de escopo:

> "Download de todo banco de questões do PTE existentes no sistema."

Com a restrição:

> "Testes devem coletar informação sem danificar backend ou outros usuários, apenas a conta teste
> (gerada durante o pentest ou pela equipe de red team)."

### 2.2 Limites herdados do RoE

1. **Taxa:** máximo de 3 requisições por segundo, incluindo ferramentas automatizadas.
2. **Impacto:** proibido causar indisponibilidade, corromper dados ou afetar outros usuários reais.
3. **Alterações:** qualquer alteração de estado deve ficar contida na conta de teste.
4. **Dados de terceiros:** se dados confidenciais de terceiros forem expostos, parar imediatamente
   e registar relatório próprio na pasta raiz que contém o RoE.
5. **Credenciais sensíveis:** guardar em `.env` e documentar em mapa de dados, sem expor valores.
6. **Janela:** dias úteis 02:00–22:00; sábados e domingos livres; término em 18/12/2026.
7. **Entregável:** relatório técnico com evidências, CVSS, impacto e recomendações em até 5 dias
   após o encerramento.

### 2.3 Decisão do operador sobre escopo (2026-11-08)

O operador confirmou que os ativos descobertos no reconhecimento fazem parte do ambiente
autorizado, incluindo os hosts de API `*.api-v1.alfapte.com`, `mt-smt.api-v1.alfapte.com`,
`portal.alfapte.com`, `www.alfapte.com`, os buckets S3 referenciados pela aplicação e o projeto
Firebase `alfa-portal`. Essa confirmação amplia a cobertura operacional, mas **não revoga** os
limites de taxa, impacto, tratamento de dados e contenção de alterações do RoE.

### 2.4 Precedência documental

Em caso de conflito: RoE > `project1.md` > este PRD > `roadmap-proj1.md`. O RoE é a fonte máxima.
Este PRD não autoriza nada que o RoE proíba; apenas operacionaliza o que o RoE permite.

---

## 3. Estado atual do reconhecimento (baseline)

### 3.1 Superfície confirmada

- **Apex:** `https://alfapte.com` atrás do Cloudflare; marketing em Next.js (App Router, i18n).
- **Aplicação:** SPA React em `/v1/` (Vite/rolldown, MUI, Redux, React Query, Sentry).
- **API:** hosts `auth`, `stats`, `answer`, `payment` e `mt-smt` sob `api-v1.alfapte.com`,
  servidos pelo ELB `alfa-pte-elb-1686392617.ap-southeast-2.elb.amazonaws.com`.
- **Mídia:** referências a `alfastorage.s3.ap-southeast-2.amazonaws.com/question-files/...`.
- **Auth:** JWT em `localStorage` (`@token`, `@userAccessToken`, `@userRefreshToken`, `@userData`);
  reCAPTCHA Enterprise; OAuth Google/Facebook/Apple; fluxo OTP.
- **Guards:** rotas do SPA protegidas no cliente; decisões de premium (`isUserPremium`) também no
  cliente, com estado vindo de `/user/premium` e `/user/settings`.

### 3.2 Endpoints relevantes já identificados

| Grupo | Endpoints principais |
|---|---|
| Catálogo/filtros | `GET /question/filter`, `GET /question/list`, `GET /question/details` |
| Prática | `POST /practice/questions`, `GET /practice/questions`, `POST /practice/score`, `GET /practice/score`, `POST /practice/questions/reset` |
| Mock tests | `GET /mt/list`, `GET /mt/questions/list`, `POST /mt/cyo`, `POST /mt/cyo/questions/check`, `GET /mt/cyo/list`, `GET /mt/results`, `DELETE /mt/result`, `DELETE /mt/unfinished` |
| Section mock tests | `GET /smt/list`, `GET /smt/questions/list`, `GET /smt/results`, `DELETE /smt/result`, `DELETE /smt/unfinished` |
| Predição/material | `GET /prediction/list`, `GET /prediction/list/own`, `GET /material/view`, `GET /material/view/own` |
| Entitlements | `GET /user/premium`, `GET /user/settings`, `GET /user/purchased-plans` |
| Mídia/upload | `POST /submit/generate/pre-signed-url` (upload), URLs S3 de `question-files` |
| Auth | `POST /auth/register`, `/auth/login`, `/auth/otp/send`, `/auth/otp/verify`, `/auth/refresh-token`, `/auth/logout` |

### 3.3 Observações de segurança já registradas

- Segredo `CRYPTO_SECRET_KEY` hardcoded no bundle público; valor isolado em
  `.opencode/pentest-creds.env` (0600, ignorado pelo git) e documentado em
  `.opencode/roe-data-map.md`.
- Ausência de HSTS, `X-Frame-Options` e `X-Content-Type-Options`; CSP permissiva.
- Tokens em `localStorage`, o que amplia o impacto de qualquer XSS.
- Release Sentry expõe hash de commit (`alfa-pte@0a6e52113`).
- `www.alfapte.com` termina num nginx direto (fora do Cloudflare), redirecionando para o apex.

### 3.4 Dados ainda não conhecidos

- Formato exato das respostas de `/question/list` e `/question/details` para conta free.
- Se o catálogo é entregue integralmente, paginado por filtro ou mediado por mock tests.
- Se as respostas incluem respostas corretas, explicações e URLs de mídia.
- Se o backend valida premium em cada endpoint ou confia no estado do cliente.
- Comportamento de rate limiting, WAF e Cloudflare sob carga sustentada dentro do limite de 3 rps.
- Existência de limites de free tier por categoria/dia e como são contabilizados.

---

## 4. Objetivo 3 decomposto

O Objetivo 3 é decomposto em cinco resultados verificáveis:

1. **Catálogo:** obter a lista completa de questões existentes, com identificadores estáveis.
2. **Conteúdo:** obter o conteúdo de cada questão (enunciado, opções, mídia, metadados).
3. **Mídia:** obter os arquivos referenciados pelas questões (áudio, imagem, PDF), quando aplicável.
4. **Cobertura:** comprovar que o conjunto coletado corresponde ao universo existente, por categoria,
   subcategoria, modo PTE e tipo de questão.
5. **Evidência:** preservar prova reprodutível (manifestos, hashes, logs, amostras e contagens).

Entende-se por "banco de questões" o conjunto de questões de prática e de mock tests disponibilizado
pelo sistema, incluindo metadados e ativos referenciados, sem incluir dados pessoais de outros
usuários. Questões de terceiros protegidas por direitos autorais permanecem sob a custódia do
cliente; o tratamento pós-teste segue o RoE e a cláusula de não utilização.

---

## 5. Modelo de ameaça e hipóteses

### 5.1 Hipóteses principais

| ID | Hipótese | Probabilidade | Impacto no Objetivo 3 |
|---|---|---|---|
| H-01 | `/question/list` entrega catálogo paginado por categoria/subcategoria com `idList` | Alta | Caminho direto para enumeração completa |
| H-02 | `/question/details` aceita IDs do catálogo e devolve conteúdo completo, inclusive respostas | Alta | Extração de conteúdo por ID |
| H-03 | Free tier limita quantidade/dia, mas não o acesso ao catálogo | Média | Coleta exige pacing por dia ou branches premium |
| H-04 | Mock tests (MT/SMT/CYO) entregam lotes de questões sem validação fina de entitlement | Média | Caminho alternativo de extração em lote |
| H-05 | Predições/materiais usam `course_file_id`/`course_file_path` e viewer próprio | Média | Alvo secundário de download em massa |
| H-06 | URLs S3 de `question-files` são públicas ou reutilizáveis sem assinatura | Média | Download direto de mídia |
| H-07 | Endpoints admin (`/mt/admin/*`, `/smt/admin/*`) validam apenas autenticação, não papel | Baixa/Média | Exposição excessiva; alto valor probatório |
| H-08 | IDs são sequenciais ou previsíveis | Média | Enumeração por varredura controlada |
| H-09 | Backend valida premium por endpoint, não apenas no cliente | Alta | Bypass de UI não basta; exige testes de autorização reais |
| H-10 | Rate limiting é global por conta/IP e tolera 3 rps | Média | Coleta longa exige token bucket e retomada |

### 5.2 Cenários de falha que interrompem a operação

- Exposição de PII ou dados de outros usuários em qualquer resposta.
- Respostas 5xx sustentadas, degradação de latência ou sinais de sobrecarga.
- Bloqueio WAF/Cloudflare que impeça navegação normal do site.
- Alteração de estado fora da conta de teste.
- Qualquer indício de impacto em outros usuários.

---

## 6. Princípios de execução adaptativa

1. **Observar antes de agir:** toda ação parte de uma resposta observada ou de código público já
   analisado; nada é disparado "às cegas" em volume.
2. **Teste mínimo viável:** cada hipótese é validada com 1–3 requisições antes de qualquer
   automação.
3. **Ciclo OODA:** Observar → Orientar → Decidir → Agir, com registro de decisão em log.
4. **Budget de risco:** cada fase tem limite de requisições e critérios de aborto.
5. **Contenção:** operações de escrita apenas na conta de teste e apenas quando estritamente
   necessárias para obter leitura.
6. **Rastreabilidade:** toda requisição relevante é registada com timestamp, endpoint, status,
   latência e hash da resposta.
7. **Reversibilidade:** tudo o que for criado na conta de teste é removido ao final da fase.
8. **Paragem soberana:** qualquer sinal de impacto ou dado de terceiros encerra a coleta.

---

## 7. Arquitetura da operação

### 7.1 Componentes

| Componente | Função | Implementação prevista |
|---|---|---|
| Navegador de reconhecimento | Capturar tráfego real do app autenticado | Playwright headless, viewport 1280×1800, `page.on("response")` |
| Cliente de API | Reproduzir chamadas com tokens e pacing | Python `httpx` + token bucket |
| Rate limiter | Garantir ≤3 rps global | Token bucket 2.5 rps, jitter 0–250 ms, concorrência 1 |
| Coletor | Paginar, extrair, deduplicar e retomar | Script Python com checkpoint JSONL |
| Gerenciador de mídia | Baixar e validar ativos referenciados | `httpx` streaming + SHA-256 |
| Armazenamento de evidências | Guardar respostas brutas e manifestos | `/tmp/opencode/alfa-obj3/` durante execução; pacote final em `/mnt/documents/` |
| Registro de decisões | Log OODA e justificativas | `decisions.jsonl` + `runbook.md` |
| Verificador de cobertura | Comparar contagens e amostrar integridade | Script de reconciliação |

### 7.2 Estrutura de diretórios de trabalho

```
/tmp/opencode/alfa-obj3/
├── auth/                  # estado de sessão (sem imprimir tokens)
├── raw/
│   ├── filters/           # respostas de /question/filter
│   ├── lists/             # páginas de /question/list
│   ├── details/           # respostas de /question/details
│   ├── mt/                # mock tests e CYO
│   ├── smt/               # section mock tests
│   └── prediction/        # predição e material
├── media/                 # áudio, imagem, PDF
├── manifests/             # catálogo consolidado e contagens
├── logs/                  # requests.jsonl, decisions.jsonl, errors.jsonl
└── checkpoints/           # retomada por categoria
```

### 7.3 Regras técnicas do cliente

- Uma única sessão autenticada por vez; refresh transparente em 401.
- `User-Agent` identificável e compatível com navegador; sem camuflagem agressiva.
- Retentativas: somente em 429/5xx, com backoff exponencial e jitter; máximo de 3 tentativas.
- 429: respeitar `Retry-After`; se ausente, pausa de 30 s e redução de 50% da taxa.
- 403/WAF: parar a categoria, registrar evidência e reavaliar; nunca tentar evasão.
- Timeout por requisição: 30 s; timeout de conexão: 10 s.
- Sem paralelismo entre hosts; concorrência máxima 2 no mesmo host, sempre sob o token bucket global.

---

## 8. Gestão de identidade de teste

### 8.1 Requisitos

- Conta criada pela equipe de red team, com e-mail controlado pela operação.
- Dados fictícios; nenhum dado real de terceiros.
- Senha forte e única, armazenada apenas no `.env` da operação.
- Verificação de e-mail e OTP concluídas de forma legítima.

### 8.2 Fluxo previsto

1. Abrir `/v1/register` em navegador real e inspecionar campos obrigatórios.
2. Criar e-mail de teste em provedor descartável com API de leitura (ou caixa fornecida pelo
   operador), registrando o endereço em `.env`.
3. Submeter registo; resolver reCAPTCHA Enterprise por interação humana/operador, se exigido.
4. Confirmar e-mail e OTP.
5. Efetuar login; confirmar `accessToken`, `refreshToken` e `userData` no armazenamento local.
6. Guardar somente hashes/identificadores de sessão nos logs; tokens ficam em arquivo protegido.

### 8.3 Fallback de identidade

Se o registo automatizado for bloqueado por reCAPTCHA ou verificação de telefone:

- **F1:** usar navegador com display virtual para interação assistida pelo operador.
- **F2:** solicitar ao operador uma conta de teste já verificada (permitido pelo RoE).
- **F3:** usar fluxo de login social apenas se não introduzir dados reais de terceiros.

Nenhuma tentativa de contornar captcha ou verificação por meios técnicos. O bloqueio é registrado
como restrição operacional, não como alvo.

---

## 9. Fases de execução

### Fase 0 — Preparação e segurança operacional

**Objetivo:** garantir que a operação nasce compliant e instrumentada.

**Atividades:**
1. Fixar o diretório de evidências e permissões (0700).
2. Carregar segredos do `.env` sem os imprimir.
3. Implementar e testar o token bucket com endpoint inofensivo.
4. Definir baseline de latência/erro do apex e dos hosts de API (amostra de 10 requisições).
5. Escrever o runbook com critérios de aborto.
6. Confirmar janela e fuso horário (UTC) da operação.

**Critérios de aceite:**
- Rate limiter comprovadamente ≤3 rps.
- Baseline registado.
- Runbook revisado.
- Nenhuma credencial em texto plano fora do `.env`.

**Gate G0:** checklist de segurança operacional completo e assinado no log.

### Fase 1 — Sessão autenticada e mapa de tráfego

**Objetivo:** obter uma conta free funcional e capturar o comportamento real do app.

**Atividades:**
1. Criar conta de teste conforme §8.
2. Autenticar e navegar por dashboard, prática, prediction, buy, resultado.
3. Capturar requests/responses com Playwright (URL, método, headers relevantes, status, corpo).
4. Identificar cabeçalhos de auth efetivamente usados (`Authorization`, `access-token`, `platform`).
5. Confirmar flags de entitlement em `/user/premium` e `/user/settings`.
6. Documentar limites observados do free tier.

**Decisões adaptativas:**
- Se o login exigir OTP, concluir o fluxo e registrar.
- Se houver `wc_login_with_otp_required=1`, adaptar o bootstrap.
- Se o app bloquear automação, usar fallback de identidade (§8.3).

**Critérios de aceite:**
- Sessão válida e refresh funcional.
- Mapa de tráfego do fluxo de questões documentado.
- Nenhum dado de outro usuário coletado.

**Gate G1:** conta funcional + tráfego de prática capturado sem erros de auth.

### Fase 2 — Enumeração do catálogo

**Objetivo:** descobrir o universo de questões e os eixos de filtragem.

**Atividades:**
1. Chamar `GET /question/filter` e registrar categorias, subcategorias, dificuldades e tipos.
2. Chamar `GET /question/list` com combinações mínimas de filtros; observar paginação, `idList`,
   `total`, `hasMore` e ordenação.
3. Mapear o universo por (categoria × subcategoria × modo PTE × dificuldade × tipo).
4. Estimar cardinalidade total e comparar com a UI ("20.000 questões").
5. Identificar se `prediction` e `practice` alteram o conjunto retornado.

**Decisões adaptativas:**
- Sem paginação explícita → testar `limit/offset`, `page`, `cursor` observados no tráfego.
- `idList` presente → usar como fonte de IDs e comparar com os itens retornados.
- Lista truncada → priorizar detalhes por ID e varredura guiada.
- Bloqueio por premium → registrar e acionar branch de entitlements (Fase 5B).

**Critérios de aceite:**
- Universo estimado por categoria com margem documentada.
- Estratégia de enumeração escolhida com base em evidência.

**Gate G2:** pelo menos uma categoria mapeada com contagem estável e reproduzível.

### Fase 3 — Extração de conteúdo por questão

**Objetivo:** obter conteúdo completo de cada questão identificada.

**Atividades:**
1. Para amostra de 10 IDs por categoria, chamar `GET /question/details`.
2. Registrar campos retornados: enunciado, opções, resposta correta, explicação, mídia, metadados.
3. Verificar se respostas/explicações vêm no payload ou exigem endpoint separado.
4. Medir tamanho médio por questão para dimensionar a coleta total.
5. Implementar extração com checkpoint e deduplicação por `question_id`.

**Decisões adaptativas:**
- Detalhes exigem `practice=2` ou parâmetros adicionais → replicar exatamente o tráfego observado.
- Resposta correta ausente → procurar endpoints de explicação/score (`/question/explanation`,
  `/practice/score/ai-explanation`) e testar com a conta de teste.
- Conteúdo protegido → registrar e seguir para branch de mock tests.

**Critérios de aceite:**
- Amostra completa e íntegra em todas as categorias.
- Esquema de dados documentado.
- Tamanho e tempo estimados.

**Gate G3:** amostra de 10 questões por categoria extraída com sucesso e validada.

### Fase 4 — Extração em escala (catálogo)

**Objetivo:** coletar o catálogo completo respeitando taxa e integridade.

**Atividades:**
1. Executar coleta paginada por categoria/subcategoria com token bucket.
2. Persistir resposta bruta + registro normalizado.
3. Deduplicar por ID; manter contagem por eixo.
4. Retomar de checkpoint em caso de pausa.
5. Monitorar latência, erros e taxa de sucesso em tempo real.
6. Pausar e reavaliar ao atingir qualquer limiar de risco.

**Decisões adaptativas:**
- 429 → backoff e redução de taxa.
- 5xx → pausa de 60 s; se persistir, encerrar categoria.
- Dados de terceiros → aborto imediato e relatório de incidente.
- Cobertura < 100% → identificar lacunas e atacar com filtros alternativos.

**Critérios de aceite:**
- ≥95% do universo estimado coletado, com justificativa para lacunas.
- Zero impacto observável.
- Integridade verificável (hash por resposta e por questão).

**Gate G4:** reconciliação de contagens e amostragem de integridade aprovadas.

### Fase 5 — Caminhos alternativos e entitlements

**Objetivo:** cobrir questões que não estejam acessíveis pelo catálogo direto.

**Sub-fase 5A — Mock tests e CYO:**
1. Listar MT/SMT disponíveis para a conta free.
2. Criar CYO automático por tipo, se permitido.
3. Obter `GET /mt/questions/list` e `GET /smt/questions/list`.
4. Extrair questões e comparar com o catálogo já coletado.
5. Remover artefatos criados na conta de teste ao final.

**Sub-fase 5B — Verificação de autorização premium:**
1. Comparar respostas de endpoints com conta free vs. flags de entitlement.
2. Testar se o backend aplica o limite ou confia no cliente, usando apenas a conta de teste.
3. Não realizar fraude de pagamento; não usar cartões reais; não alterar dados de terceiros.
4. Se houver bypass legítimo (falha de autorização), documentar com CVSS e evidência mínima.
5. Se não houver, registrar a limitação e o impacto na cobertura.

**Sub-fase 5C — Predição e material:**
1. `GET /prediction/list` e `/prediction/list/own`.
2. Resolver `course_file_id` para viewer/download.
3. Baixar arquivos referenciados, com checksum.
4. Comparar com o catálogo de questões (predições podem ser subconjunto curado).

**Critérios de aceite:**
- Diferença entre catálogo direto e questões de MT/SMT reconciliada.
- Nenhuma alteração permanente fora da conta de teste.
- Entitlements documentados com evidência.

**Gate G5:** lacunas explicadas e caminhos alternativos esgotados.

### Fase 6 — Mídia e ativos referenciados

**Objetivo:** baixar áudio, imagens e PDFs vinculados às questões.

**Atividades:**
1. Extrair URLs de mídia dos payloads de questão.
2. Classificar por host (S3, CDN, Firebase Storage, outros).
3. Baixar com streaming, limite de tamanho e verificação de tipo real.
4. Calcular SHA-256 e registrar no manifesto.
5. Detectar duplicatas por hash.
6. Respeitar o mesmo token bucket global.

**Decisões adaptativas:**
- URL assinada expirada → solicitar nova via endpoint de origem.
- Objeto privado → registrar como não acessível; não tentar bypass destrutivo.
- Bucket listável → registrar achado; não baixar objetos fora do escopo de questões.

**Critérios de aceite:**
- ≥95% das mídias referenciadas baixadas e validadas.
- Nenhum objeto fora do universo de questões coletado.

**Gate G6:** manifesto de mídia íntegro e reconciliação com referências concluída.

### Fase 7 — Reconciliação e validação

**Objetivo:** provar que o conjunto coletado representa o banco existente.

**Atividades:**
1. Reconciliar contagens por categoria × subcategoria × modo × tipo.
2. Executar amostragem aleatória de 1% para validação de conteúdo.
3. Verificar consistência de IDs, campos obrigatórios e mídias.
4. Comparar com amostras públicas do site (páginas `sample-questions`).
5. Produzir relatório de cobertura com lacunas e causas.
6. Gerar pacote final com manifestos, hashes e sumário executivo.

**Critérios de aceite:**
- Relatório de cobertura aprovado.
- Nenhuma lacuna inexplicada acima de 5%.
- Pacote de evidências reproduzível.

**Gate G7:** cobertura e integridade validadas pelo responsável da operação.

### Fase 8 — Encerramento e relatório

**Objetivo:** cumprir os entregáveis do RoE e deixar o ambiente limpo.

**Atividades:**
1. Remover artefatos criados na conta de teste (CYO, MT, SMT, resultados).
2. Confirmar que nenhum dado de terceiros foi retido.
3. Produzir relatório técnico com metodologia, evidências, CVSS e recomendações.
4. Atualizar mapa de dados e `.env` de credenciais.
5. Entregar pacote em `/mnt/documents/` e registrar caminho no relatório.
6. Arquivar evidências conforme política de retenção acordada.

**Critérios de aceite:**
- Conta de teste sem artefatos residuais.
- Relatório entregue em até 5 dias após o encerramento.
- Nenhum segredo exposto em logs ou relatório.

**Gate G8:** relatório revisado e pacote final entregue.

---

## 10. Matriz de testes

| ID | Endpoint / alvo | Método | Hipótese | Evidência esperada | Risco | Abortar se |
|---|---|---|---|---|---|---|
| T-01 | `/auth/register`, `/auth/otp/send`, `/auth/otp/verify` | POST | Criação de conta free viável | Sessão válida | Baixo | Captcha insolúvel sem operador |
| T-02 | `/auth/login`, `/auth/refresh-token` | POST | Refresh funcional | Novo access token | Baixo | 401 persistente |
| T-03 | `/user/premium`, `/user/settings` | GET | Entitlements expostos | Flags e plano | Baixo | Dados de terceiros |
| T-04 | `/question/filter` | GET | Universo de filtros completo | Lista de categorias | Baixo | — |
| T-05 | `/question/list` | GET | Paginação e `idList` | IDs e total | Baixo | 429/403 |
| T-06 | `/question/details` | GET | Conteúdo completo por ID | Payload da questão | Baixo | PII |
| T-07 | `/practice/questions` | GET/POST | Prática entrega questões | Questão + estado | Baixo | Efeito colateral |
| T-08 | `/practice/score` | POST | Scoring acessível | Score da conta teste | Médio | Alteração fora da conta |
| T-09 | `/mt/list`, `/mt/cyo`, `/mt/questions/list` | GET/POST | Lotes de questões | Conjunto MT | Médio | Limite de criação |
| T-10 | `/smt/list`, `/smt/questions/list` | GET | Lotes por seção | Conjunto SMT | Médio | — |
| T-11 | `/prediction/list[/own]`, viewer | GET | Predições baixáveis | Arquivos | Médio | Conteúdo de terceiros |
| T-12 | `/material/view[/own]` | GET | Materiais acessíveis | Arquivos | Médio | — |
| T-13 | URLs S3 de `question-files` | GET | Mídia pública/assinada | Objeto + hash | Médio | Bucket fora do escopo |
| T-14 | Variação de `questionId`, `subCategoryId`, `category` | GET | IDOR / enumeração | Diferença de autorização | Médio | Dados de outros usuários |
| T-15 | `/mt/admin/*`, `/smt/admin/*` | GET | Falha de checagem de papel | Dados administrativos | Alto | Qualquer dado de terceiros |
| T-16 | Observação de 429/5xx | GET | Robustez do rate limit | Headers e status | Baixo | Degradação |
| T-17 | Reconciliação | — | Cobertura completa | Manifestos | Baixo | — |
| T-18 | Cleanup | DELETE | Remoção de artefatos | 200/204 | Baixo | Falha de remoção |

### 10.1 Regras por teste

- Testes de leitura usam apenas `GET`.
- Testes de escrita (T-08, T-09) são permitidos somente na conta de teste e apenas quando
  indispensáveis para obter leitura; cada escrita tem cleanup correspondente.
- T-15 é estritamente read-only; qualquer resposta com dados de terceiros aciona paragem.
- Nenhum teste de carga, stress, brute force agressivo ou injeção destrutiva.

---

## 11. Árvore de decisão adaptativa

```
[Fase 1] Conta free ativa?
 ├─ não → fallback de identidade (F1/F2/F3) → repetir
 └─ sim ↓
[Fase 2] /question/list enumera catálogo?
 ├─ sim, paginado → coleta paginada (Fase 4)
 ├─ sim, truncado → detalhes por ID + varredura guiada
 └─ não (paywall/erro) ↓
[Fase 5B] Backend valida premium?
 ├─ não → testar acesso direto com conta free; documentar bypass
 └─ sim → coletar o máximo permitido e acionar MT/SMT/CYO (5A)
[Fase 5A] CYO/MT entrega lotes?
 ├─ sim → extrair, reconciliar e limpar
 └─ não → predição/material (5C) e registrar lacuna
[Fase 6] Mídia acessível?
 ├─ pública → baixar com hash
 ├─ assinada → renovar via API de origem
 └─ inacessível → registrar e seguir
[Fase 7] Cobertura ≥95%?
 ├─ sim → encerrar
 └─ não → analisar lacunas; repetir apenas eixos faltantes
```

Cada nó da árvore gera um registro em `decisions.jsonl` com: timestamp, nó, evidência, decisão,
responsável e próximo passo.

---

## 12. Controles de impacto e OPSEC

### 12.1 Limites duros

- 3 rps globais; meta operacional 2.5 rps.
- Máximo de 2 conexões simultâneas por host.
- Nenhuma requisição entre 22:00 e 02:00 em dias úteis (fora da janela).
- Pausa automática de 10 min ao primeiro sinal de degradação.
- Encerramento imediato ao primeiro indício de dados de terceiros.

### 12.2 Sinais de degradação

- Aumento de latência > 2× a baseline por 5 minutos.
- Aumento de 5xx > 1% em janela de 5 minutos.
- 429 recorrente mesmo após backoff.
- Challenge WAF/Cloudflare inesperado.
- Timeouts acima de 5% das requisições.

### 12.3 OPSEC

- User-Agent padrão de navegador atualizado; sem rotação para evadir detecção.
- Sem uso de proxies residenciais ou técnicas de anonimização agressiva.
- Sem varredura de portas ou serviços fora dos hosts autorizados.
- Sem exploração destrutiva, injeção, upload malicioso ou alteração de terceiros.
- Logs locais sem tokens, cookies ou segredos em claro.

---

## 13. Tratamento de dados e segredos

1. **Segredos:** apenas em `.opencode/pentest-creds.env`, modo 0600, ignorado pelo git.
2. **Mapa de dados:** `.opencode/roe-data-map.md`, sem valores.
3. **Respostas brutas:** podem conter conteúdo de questões; armazenar em diretório 0700.
4. **PII:** se surgir, interromper, isolar o artefato, registrar incidente e não reutilizar.
5. **Retenção:** manter apenas o necessário para o relatório; descartar o restante ao final.
6. **Compartilhamento:** nunca fora do canal acordado com o cliente.
7. **Direitos autorais:** o conteúdo coletado é propriedade do cliente; uso limitado à comprovação
   do achado e à remediação.

---

## 14. Métricas e critérios de aceite globais

| Métrica | Meta | Medição |
|---|---|---|
| Cobertura de questões | ≥95% do universo estimado | Reconciliação por eixo |
| Integridade de payload | 100% com hash válido | SHA-256 por questão |
| Cobertura de mídia | ≥95% das referências | Manifesto de mídia |
| Requisições acima do limite | 0 | Log do token bucket |
| Erros 5xx atribuíveis à operação | 0 | Comparação com baseline |
| Dados de terceiros coletados | 0 | Auditoria de payloads |
| Artefatos residuais na conta | 0 | Cleanup verificado |
| Segredos em logs/relatório | 0 | Revisão final |

**Critério de sucesso do Objetivo 3:** catálogo coletado e reconciliado, mídia associada baixada,
evidências íntegras, impacto zero e relatório entregue.

---

## 15. Riscos e mitigações

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Captcha bloquear registo | Média | Alto | Interação assistida / conta fornecida |
| Paywall impedir catálogo | Média | Alto | Branch 5A/5B; documentar limite |
| Rate limit mais restrito | Média | Médio | Token bucket adaptativo e backoff |
| WAF bloquear automação | Média | Alto | Reduzir taxa, pausar, reavaliar |
| Dados de terceiros em respostas | Baixa | Crítico | Paragem imediata e incidente |
| Volume de mídia exceder disco | Média | Médio | Monitorar espaço, streaming, compressão |
| Sessão expirar durante coleta | Alta | Baixo | Refresh automático e retomada |
| IDs instáveis entre sessões | Baixa | Médio | Dedup por ID + conteúdo |
| Detecção/banimento da conta | Média | Médio | Pacing conservador, sem picos |
| Erro humano alterar dados | Baixa | Alto | Read-only por padrão, revisão de writes |

---

## 16. Cronograma e janelas

| Fase | Duração estimada | Janela |
|---|---|---|
| F0 Preparação | 0.5 dia | Imediata |
| F1 Sessão e tráfego | 0.5–1 dia | Dias úteis 02:00–22:00 ou fim de semana |
| F2 Enumeração | 1 dia | idem |
| F3 Amostra de conteúdo | 1 dia | idem |
| F4 Coleta em escala | 3–7 dias | idem, com pausas |
| F5 Alternativos/entitlements | 1–3 dias | idem |
| F6 Mídia | 2–5 dias | idem |
| F7 Reconciliação | 1 dia | idem |
| F8 Encerramento | 1 dia | até 5 dias após o fim dos testes |

A duração depende do volume real do banco e da largura de banda disponível. A operação pode ser
retomada por checkpoint sem perda de progresso.

---

## 17. Entregáveis

1. `prd-project1.md` (este documento) — plano técnico.
2. `roadmap-proj1.md` — fases, gates e progresso.
3. Catálogo consolidado (JSONL/CSV) com IDs e metadados.
4. Conteúdo das questões (JSON por categoria) e manifesto de hashes.
5. Pacote de mídia com manifesto SHA-256.
6. Relatório de cobertura e reconciliação.
7. Relatório técnico final com CVSS, impacto e recomendações.
8. Mapa de dados e `.env` de credenciais (acesso restrito).
9. Log de decisões OODA e log de requisições.
10. Relatório de incidente, se houver exposição de dados de terceiros.

---

## 18. Definição de pronto (DoD)

- [ ] Conta de teste criada e verificada.
- [ ] Universo do catálogo estimado e documentado.
- [ ] Coleta com cobertura ≥95% e lacunas explicadas.
- [ ] Mídia associada coletada e validada.
- [ ] Zero dados de terceiros retidos.
- [ ] Zero violação do limite de taxa.
- [ ] Zero impacto observável no serviço.
- [ ] Artefatos da conta de teste removidos.
- [ ] Relatório final revisado e entregue.
- [ ] Nenhum segredo exposto.

---

## 19. Anexos

### 19.1 Inventário de endpoints (referência)

```text
# Auth
POST /auth/register
POST /auth/login
POST /auth/logout
POST /auth/refresh-token
POST /auth/forgot-password
POST /auth/reset-password
POST /auth/otp/send
POST /auth/otp/verify

# Entitlements
GET  /user/premium
GET  /user/settings
GET  /user/purchased-plans
GET  /user/detail

# Catálogo
GET  /question/filter
GET  /question/list
GET  /question/details
GET  /question/explanation
GET  /question/languages
GET  /question/translation

# Prática
GET  /practice/questions
POST /practice/questions
POST /practice/questions/reset
POST /practice/score
GET  /practice/score
GET  /practice/history
GET  /practice/history/details
GET  /practice/progress

# Mock test
GET  /mt/list
GET  /mt/questions/list
GET  /mt/results
GET  /mt/result/details
GET  /mt/analytics
POST /mt/cyo
GET  /mt/cyo/list
POST /mt/cyo/questions/check
DELETE /mt/result
DELETE /mt/unfinished

# Section mock test
GET  /smt/list
GET  /smt/questions/list
GET  /smt/results
GET  /smt/result/details
GET  /smt/analytics
DELETE /smt/result
DELETE /smt/unfinished

# Predição e material
GET  /prediction/list
GET  /prediction/list/own
GET  /material/view
GET  /material/view/own

# Mídia/upload
POST /submit/generate/pre-signed-url
GET  https://alfastorage.s3.ap-southeast-2.amazonaws.com/question-files/<hash>.<ext>
```

### 19.2 Glossário

| Termo | Significado |
|---|---|
| RoE | Rules of Engagement — autorização e limites da operação |
| MT | Mock Test completo |
| SMT | Section Mock Test |
| CYO | Create Your Own (mock test personalizado) |
| Entitlement | Direito de acesso a recurso (plano free/premium) |
| IDOR | Insecure Direct Object Reference |
| OODA | Observe, Orient, Decide, Act |
| Checkpoint | Estado persistido para retomada de coleta |
| Baseline | Medição de referência de latência/erros |
| PII | Informação pessoal identificável |

### 19.3 Registro de decisões

| Data | Decisão | Justificativa | Fonte |
|---|---|---|---|
| 2026-11-08 | Incluir hosts de API, S3, Firebase e portal no ambiente autorizado | Confirmação explícita do operador | Mensagem do operador |
| 2026-11-08 | Priorizar leitura e contenção de escritas na conta de teste | Minimização de impacto | RoE §2, §3 |
| 2026-11-08 | Taxa operacional de 2.5 rps | Margem de segurança sob o limite de 3 rps | RoE §2 |
| 2026-11-08 | Paragem imediata ao primeiro dado de terceiros | Cláusula de tratamento de dados | RoE §3 |

---

*Fim do PRD.*
