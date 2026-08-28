# 🛡️ Diretivas de Grupo (GPOs) e Políticas de Segurança

Este documento detalha as Group Policy Objects (GPOs) aplicadas no domínio para garantir a segurança e padronização do ambiente.

---
1. `GPO-01: Sec_BlockRemovableMedia`
* *Escopo / Alvo:* `OU=Financeiro`
* *Ação:* Bloqueio total de leitura e escrita em dispositivos de armazenamento removível (USB / Pendrives) para conter riscos de vazamento de dados.
* * **Motivação:** Prevenção contra vazamento de dados confidenciais e infecção por malwares.
#### Evidência de Validação
![Print do bloqueio no cliente Windows 11](../assets/evidencia-gpo-01.png)

---
2. `GPO-02: Sec_PasswordPolicy_Tiering`
* *Escopo / Alvo:* Raiz do Domínio / `OU=Admins`
* *Ação:* Exigência de comprimento mínimo de 14 caracteres, histórico de 24 senhas e bloqueio automático de conta após 5 tentativas incorretas.
* **Motivação:** Mitigação de ataques de força bruta (*Password Spraying*).
#### Evidência de Validação
![Print do relatório do GPMC / Resultant Set of Policy](../assets/evidencia-gpo-02.png)

---

3. `GPO-03: Env_DriveMapping_Dept`
* *Escopo / Alvo:* `OU=Empresa`
* *Ação:* Mapeamento dinâmico da unidade de rede S: apontando para o diretório específico do departamento do usuário logado. 
* **Motivação:** Padronização do acesso aos recursos de rede, facilidade de navegação para o usuário final e centralização de backups do servidor de arquivos.
#### Evidência de Validação
![Print do File Explorer no Windows 11 mostrando a unidade S: mapeada](../assets/evidencia-gpo-03.png)

---
# ⬇️ **NAVEGAÇÃO:**

↩️ **[Retornar ao Projeto](https://github.com/rodrigolsoares-infra/suporte-infra-hibrida)**

➡️ **[Conferir secção sobre os Scripts](https://github.com/rodrigolsoares-infra/suporte-infra-hibrida/blob/main/docs/listar-scripts.md)**
 
👤 **[Ir para o Portfólio do Meu Perfil](https://github.com/rodrigolsoares-infra)**
 
---
