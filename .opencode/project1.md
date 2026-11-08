atingir o objetivo 3 do Roe (roe disponivel em /.opencode/pte-Termo_Autorizacao_Pentest_ROE_001 (2) - Copia)

---

## Instruções operacionais permanentes (operador)

1. Guardar todos os artefactos da operação dentro de `/dev-server`, preferencialmente em `/dev-server/.opencode/`.
2. Ao concluir cada fase que produza novos ficheiros, criar um backup completo de `/dev-server` (exceto o runtime do OpenCode e o próprio diretório de backups) em `/dev-server/.opencode/backups/`.
3. Manter segredos e credenciais apenas em `.opencode/pentest-creds.env` (0600, fora do git).
