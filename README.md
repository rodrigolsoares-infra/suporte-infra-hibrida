# 🚀 Projeto: Gestão Híbrida de Identidades com AD DS, PowerShell e Microsoft Entra ID

> **Escopo do Projeto:** Laboratório focado em automação de identidades via PowerShell, aplicação de políticas de segurança por GPO, delegação de privilégios e sincronização de identidade híbrida com Microsoft Entra ID.

## 🎓 Fundamentação Teórica & Competências Exercitadas

Este laboratório foi projetado para aplicar na prática os conceitos teóricos desenvolvidos em formações profissionais e documentações oficiais de arquitetura em nuvem e infraestrutura:

* **Fundamentos de Suporte em TI (Certificação Profissional Google IT Support):**
  * Serviços de infraestrutura de rede (DNS, DHCP, endereçamento IP e sub-redes).
  * Administração de sistemas de arquivos, permissões NTFS e compartilhamento de rede.
  * Resolução de problemas (*troubleshooting*) de conectividade cliente-servidor.

* **Conceitos Fundamentais de Nuvem (Microsoft Azure Fundamentals / AZ-900):**
  * Modelos de Identidade Híbrida e autenticação centralizada (Single Sign-On - SSO).
  * Gerenciamento de identidades, acesso e governança no **Microsoft Entra ID**.
  * Entendimento dos modelos de serviço em nuvem (SaaS, PaaS, IaaS) e sincronização com infraestrutura *on-premises*.

* **Administração de Sistemas Windows Server & Automação:**
  * Estruturação de serviços de diretório (**AD DS**), arquitetura de OUs e grupos de segurança no modelo AGDLP.
  * Gerenciamento e aplicação de políticas de grupo (**GPO**) para endurecimento (*hardening*) de estações.
  * Automação de tarefas administrativas e manipulação de objetos em massa via **PowerShell**.
  
---
## 1. Configurações Base do Ambiente
Abaixo estão os parâmetros essenciais definidos na infraestrutura local e em nuvem para sustentar o ambiente híbrido de identidades.
### 1.1. Active Directory Domain Services (AD DS - Local)

| Parâmetro | Configuração Aplicada |
| :--- | :--- |
| **FQDN do Domínio** | `techcorp.loca` (com sufixo UPN alternativo `techcorp.com`) |
| **NetBIOS** | `TECHCORP` |
| **Sistema Operacional DC** | Windows Server 2022 Standard |
| **Serviços Ativos** | AD DS, DNS Server, DHCP Server |
| **Endereçamento IP (DC)** | `192.168.1.10/24` (Estático) |

<Details>
📷 Evidência: (Insira um print comprovando o ip, dns e domínio local).<br>
📷 Evidência: (Insira um print comprovando o ip, dns e domínio local).<br>
📷 Evidência: (Insira um print comprovando o ip, dns e domínio local).
</Details>
### 1.2. Tenant no Microsoft Entra ID (Nuvem)
* **Tenant ID / Domínio:** `techcorp.onmicrosoft.com`
* **Domínio Personalizado:** `techcorp.com` (Validado via registro TXT no DNS)
* **Licenciamento de Teste:** Microsoft 365 E5 / Entra ID P1
### 1.3. Agente de Integração (Microsoft Entra Connect)
* **Método de Autenticação:** Password Hash Synchronization (PHS)
* **Recursos de Segurança:** Password Writeback (sincronização bidirecional de alterações de senha)
* **Filtro de Sincronização:** Baseado em Unidades Organizacionais (OU-Based Filtering)

---
## 2. Topologia e Arquitetura do Ambiente
```plaintext
[ ON-PREMISES INFRASTRUCTURE ]                       [ MICROSOFT CLOUD SERVICES ]
Subnet: 192.168.1.0/24                               Tenant: techcorp.onmicrosoft.com
+-------------------------------------+             +---------------------------------+
|  Windows Server 2022 (Domain Ctrl)  |             |       Microsoft Entra ID        |
|  Hostname: DC01                     |             |       (Cloud Identity)          |
|  IP: 192.168.1.10                   |             |       Domain: techcorp.com      |
|  Roles: AD DS, DNS, DHCP            |             +---------------------------------+
+-------------------------------------+                              ^
                   |                                                 |
                   | (Internal Traffic)                              | Sync HTTPS / Port 443
                   v                                                 | (Password Hash Sync)
+-------------------------------------+                              |
|  Microsoft Entra Connect Server     |                              |
|  (Sync Agent / Identity Bridge)     |------------------------------+
+-------------------------------------+
                   |
                   | GPO / Kerberos / NTLM
                   v
+-------------------------------------+
|  Windows 11 Client (Workstation)    |
|  Hostname: CLI01                    |
|  IP: 192.168.1.50 (DHCP)            |
|  OU: OU=TI,OU=Empresa_TECHCORP      |
|  User: `rodrigo.soares@techcorp.com`|
+-------------------------------------+
```

<!-- ![Topologia da Arquitetura Híbrida](./docs/topologia-arquitetura.png) -->

---
## 3. Automação com Scripts:
#### *Lista de Scripts validados em aplicação:

1. Provisionamento massivo de usuários (PowerShell):*
*Função:* Para eliminar tarefas manuais e padronizar a administração do ambiente, foram desenvolvidos scripts automatizados para gerenciamento de usuários e grupos.
* **Script em Destaque:** [`scripts/ProvisionaUsuarios.ps1`](./scripts/ProvisionaUsuarios.ps1) — Provisionamento em massa via CSV.
* **Base de Dados:** [`scripts/usuarios.csv`](./scripts/usuarios.csv)

👉 **[Consulte o Acervo Completo de Scripts e Evidências](./docs/listar-scripts.md)**

---

## 4. Políticas de Grupo e Segurança (GPOs)

A estrutura de GPOs foi desenhada para aplicar o princípio de menor privilégio e garantir o endurecimento (*hardening*) das estações de trabalho.
#### *Lista de GPOs validados em aplicação:*

1. `GPO-01: Sec_BlockRemovableMedia`
* *Escopo / Alvo:* `OU=Financeiro`
* *Ação:* Bloqueio total de leitura e escrita em dispositivos de armazenamento removível (USB / Pendrives) para conter riscos de vazamento de dados.

2. `GPO-02: Sec_PasswordPolicy_Tiering`
* *Escopo / Alvo:* Raiz do Domínio / `OU=Admins`
* *Ação:* Exigência de comprimento mínimo de 14 caracteres, histórico de 24 senhas e bloqueio automático de conta após 5 tentativas incorretas.

3. `GPO-03: Env_DriveMapping_Dept`
* *Escopo / Alvo:* `OU=Empresa`
* *Ação:* Mapeamento dinâmico da unidade de rede S: apontando para o diretório específico do departamento do usuário logado. 

👉 **[Consulte a Lista Completa de GPOs e Diretivas Aplicadas](./docs/listar-gpos.md)**

---
## 5. Conceitos de Segurança Realistas Aplicados

* *Modelo de Menor Privilégio (Least Privilege):* Separação entre contas nominais corporativas e contas de administração de serviços.

* *Delegação de Controle (Delegation of Control):* Configuração do grupo `GRP_Suporte_N1` com permissão exclusiva de reset de senha e desbloqueio de contas na `OU=Usuarios`, sem conceder privilégios de Domain Admin.

* *Administração em Camadas (Tiering Model):* Restrição do uso de contas com privilégios elevados em estações de trabalho comuns.

<Details>
📷 Evidência: (Insira um print do painel com as unidades organizacionais e grupos).
</Details>

## 6. Integração Híbrida e Validação (Entra Connect)

1. Escopo de Sincronização: Configurado filtro por OUs para garantir que apenas contas corporativas ativas na `OU=Empresa_TECHCORP` sejam sincronizadas para a nuvem.

2. Mapeamento UPN: Garantida correspondência do sufixo UPN local com o domínio validado na nuvem (`@techcorp.com`).

3. Validação dos Resultados:
* Local: Contas provisionadas via PowerShell verificadas no Active Directory local.
* Nuvem: Confirmação no Portal do Microsoft Entra ID com atributo `Directory synced: Yes` ativo.

<Details>
📷 Evidência: (Insira um print do portal do Entra ID mostrando os usuários sincronizados).
</Details>

---
## Planejamento do Projeto:
```plaintext
📁 projeto-gestao-identidades-hibrida/
├── 📄 README.md.            <-- (Projeto completo)
├── 📁 scripts/
│   ├── 📜 ProvisionUsers.ps1 <-- (Scripts PS)
│   └──📃 users.csv              <-- (Arquivos de dados)
├── 📁 docs/
│    ├──📄 listar-scripts.md.<-- (Scripts em detalhes)
│    └──📄 listar-gpos.md    <-- (GPOs em detalhes)
└── 📁 assets/
    └── 📸 ad-ds.png         <-- (Print das evidências)
```
---
## 👤 Autor
**Rodrigo Luiz Soares**
* **LinkedIn:** [linkedin.com/in/rodrigolzsoares](https://linkedin.com)
* **Certificações Concluídas/Em Andamento:** Google IT Support | AZ-900 Microsoft Azure Fundamentals | Em preparação Comptia Network+
---
---
⚠️ **NAVEGAÇÃO:**

➡️ **[Conferir secção sobre as GPOs](./docs/listar-gpos.md)**

➡️ **[Conferir secção sobre as GPOs](./docs/listar-gpos.md)**
 
👤 **[Ir para o Portfólio do Meu Perfil](https://github.com/rodrigolsoares-infra)**

---
