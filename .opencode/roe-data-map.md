# Mapa de Dados — PENTEST-ROE-001

**Alvo:** https://alfapte.com/**
**Data:** 2026-11-08
**Fase:** Reconhecimento (mapeamento de superfície)
**Fonte primária:** bundle público do app em `https://alfapte.com/v1/assets/`

Valores sensíveis ficam apenas em `pentest-creds.env` (mode 0600, ignorado pelo git).

## Credenciais e segredos

| ID | Nome | Classificação | Local | Contexto de exposição | Impacto potencial | Tratamento |
|----|------|---------------|-------|-----------------------|-------------------|------------|
| C-01 | `CRYPTO_SECRET_KEY` | Confidencial | `helpers-Ba0AkfBo.js` (config `A`) | Chave estática embutida no JavaScript servido a qualquer visitante; usada no fluxo legado do site antigo (`OLD_SITE_SERVER`) | Se proteger dados/URLs legadas, permite forjar ou decifrar valores sem autenticação | Valor em `pentest-creds.env`; reportar na fase de relatório |

## Identificadores públicos (não secretos, mas úteis)

| ID | Nome | Observação |
|----|------|------------|
| P-01 | `FIREBASE_API_KEY` + projeto `alfa-portal` | Chave pública de cliente Firebase; expõe project ID, auth domain, storage bucket `alfa-portal.firebasestorage.app` |
| P-02 | `SENTRY_DSN` + release `alfa-pte@0a6e52113` | DSN público de cliente; release revela hash de commit do deploy |
| P-03 | `RAZOR_PAY_KEY` (`rzp_live_…`) | Apenas key id público do Razorpay (não é a secret key) |
| P-04 | reCAPTCHA site keys (login e registo) e Google OAuth client ID | Públicos por desenho |
| P-05 | Facebook App ID / Apple bundle `com.alfapte.alfapte` | Públicos por desenho |

## Notas de manuseamento

- Não copiar valores para relatórios, chat, tickets ou commits.
- O ficheiro `.env` está excluído via `.gitignore` (`.opencode/*.env`).
- Qualquer uso dos segredos deve limitar-se a comprovar impacto dentro dos objetivos do RoE.
