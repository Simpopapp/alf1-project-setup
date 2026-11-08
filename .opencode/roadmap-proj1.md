# Roadmap — Objetivo 3 · PENTEST-ROE-001

Fonte: `project1.md` → `prd-project1.md` → este roadmap.
Regra: só marcar `[x]` quando os gates da fase passarem, com evidência registrada.

## Fases

- [ ] **F0 — Preparação e segurança operacional**
  - Evidências: token bucket testado, baseline de latência, runbook, `.env` 0600.
  - Gates: limite ≤3 rps comprovado · baseline registado · runbook aprovado · zero segredos fora do `.env`.

- [ ] **F1 — Sessão autenticada e mapa de tráfego**
  - Evidências: conta de teste ativa, captura Playwright do fluxo de prática, flags de entitlement.
  - Gates: login + refresh funcionais · tráfego de questões mapeado · nenhum dado de terceiros.

- [ ] **F2 — Enumeração do catálogo**
  - Evidências: `/question/filter`, páginas de `/question/list`, universo estimado por eixo.
  - Gates: ≥1 categoria com contagem estável · estratégia de enumeração escolhida por evidência.

- [ ] **F3 — Extração de conteúdo por questão**
  - Evidências: amostra de 10 IDs por categoria via `/question/details`; esquema de dados.
  - Gates: amostra íntegra em todas as categorias · tamanho/tempo estimados · campos obrigatórios validados.

- [ ] **F4 — Coleta em escala (catálogo)**
  - Evidências: JSONL bruto + normalizado, checkpoints, dedup, logs de taxa.
  - Gates: ≥95% do universo estimado · zero impacto observável · hashes de integridade válidos.

- [ ] **F5 — Alternativos e entitlements (MT/SMT/CYO, premium, predição/material)**
  - Evidências: lotes MT/SMT extraídos, diff vs catálogo, testes de autorização, downloads de predição.
  - Gates: lacunas reconciliadas · escritas só na conta de teste · entitlements documentados.

- [ ] **F6 — Mídia e ativos referenciados**
  - Evidências: arquivos baixados, manifesto SHA-256, classificação por host.
  - Gates: ≥95% das referências · nenhum objeto fora do universo de questões.

- [ ] **F7 — Reconciliação e validação**
  - Evidências: relatório de cobertura, amostragem de 1%, comparação com páginas públicas.
  - Gates: cobertura e integridade validadas · lacunas ≤5% ou justificadas · pacote reproduzível.

- [ ] **F8 — Encerramento e relatório**
  - Evidências: cleanup da conta, relatório técnico com CVSS, mapa de dados, pacote final.
  - Gates: zero artefatos residuais · relatório revisado e entregue · zero segredos expostos.

## Progresso

- Fases concluídas: 0/9
- Próxima fase: F0 — Preparação e segurança operacional
- Bloqueios: nenhum
