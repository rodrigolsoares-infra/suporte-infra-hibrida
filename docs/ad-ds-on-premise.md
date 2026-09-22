# 🏛️ Infraestrutura Corporativa On-Premises (TechCorp Local)

Este repositório documenta a arquitetura, implantação, automação e governança da infraestrutura de TI *On-Premises* para a empresa fictícia **TechCorp**, estruturada sobre a plataforma **Microsoft Windows Server 2022** e virtualizada via **Hyper-V**.

---

## 📌 Capítulo 1: Arquitetura de Rede e Topologia On-Premises

### 1.1 Visão Geral da Rede
A infraestrutura local foi projetada em uma rede corporativa isolada (`192.168.10.0/24`), seguindo o padrão de nomenclatura estroncado para servidores e estrito controle de endereçamento IP.

| Hostname | Função / Serviços Executados | Endereço IP | Máscara / Gateway | SO / Detalhes |
| :--- | :--- | :--- | :--- | :--- |
| **`DC01`** | Domain Controller (AD DS), DNS Server, DHCP Server | `192.168.10.10` | `/24` (192.168.10.1) | Windows Server 2022 |
| **`DC02`** | *(Reservado para Réplica de AD / Alta Disponibilidade)* | `192.168.10.11` | `/24` (192.168.10.1) | *(Reservado)* |
| **`FS-01`** | File Server Dedicado (Compartilhamento SMB / ABE / AGDLP) | `192.168.10.12` | `/24` (192.168.10.1) | Windows Server 2022 |
| **`MON-01`**| Servidor de Monitoramento de Infraestrutura | `192.168.10.13` | `/24` (192.168.10.1) | *(Em planejamento)* |
| **`CLI-01`**| Estação de Trabalho do Usuário | DHCP (`192.168.10.50+`) | `/24` (192.168.10.1) | Windows 11 Enterprise |

### 1.2 Configuração de Serviços Nativos de Rede
* **DNS Server (`DC01`):** Zona Primária de Resolução Direta `techcorp.local` integrada ao Active Directory. Configuração de *Forwarders* para os servidores DNS do Google (`8.8.8.8` e `8.8.4.4`) para resolução de nomes externos.
* **DHCP Server (`DC01`):** Escopo de distribuição automática de IPs para estações (`192.168.10.50` a `192.168.10.200`), distribuindo a opção `003 Router` (`192.168.10.1`) e opção `006 DNS Server` (`192.168.10.10`).

---

## 🌳 Capítulo 2: Active Directory Domain Services (AD DS)

### 2.1 Estrutura de Unidades Organizacionais (OUs)
Para garantir a aplicação organizada de políticas de grupo (GPOs) e delegação administrativa, foi criada a seguinte estrutura hierárquica abaixo da raiz do domínio `techcorp.local`:

```text
techcorp.local
 └── TechCorp_Company
      ├── 📁 Administrative
      │    ├── 🖥️ Domain_Controllers
      │    └── 🖥️ Servers
      ├── 📁 Departments
      │    ├── 📁 Comercial
      │    ├── 📁 Diretoria
      │    ├── 📁 Financeiro
      │    ├── 📁 RH
      │    └── 📁 TI
      ├── 📁 Groups (Grupos Globais e Domain Local)
      └── 📁 Workstations (Computadores do domínio)
      ```

2.2 Automação no Provisionamento de Usuários (PowerShell)
A carga inicial de contas de usuários foi automatizada via PowerShell, lendo uma lista de colaboradores formatada em arquivo .csv e populando os atributos necessários (Nome, Sobrenome, UPN, Departamento e Unidade Organizacional de destino):

```powershell
# Ingestão de usuários a partir de CSV e criação no AD DS
Import-Csv -Path "C:\AdminScripts\Users_TechCorp.csv" -Delimiter ";" | ForEach-Object {
    $Password = ConvertTo-SecureString "SenhaTemp@2026" -AsPlainText -Force
    $UPN = "$($_.SamAccountName)@techcorp.com"
    
    New-ADUser `
        -Name "$($_.FirstName) $($_.LastName)" `
        -GivenName $_.FirstName `
        -Surname $_.LastName `
        -SamAccountName $_.SamAccountName `
        -UserPrincipalName $UPN `
        -Path $_.OU `
        -Department $_.Department `
        -AccountPassword $Password `
        -Enabled $true `
        -ChangePasswordAtLogon $true
}
```
2.3 Atributos de Proteção e Resiliência
Lixeira do Active Directory (AD Recycle Bin): Ativada via PowerShell no nível de funcionalidade da floresta para permitir a recuperação instantânea de objetos excluídos acidentalmente sem necessidade de restauração de backup (Authoritative Restore).

UPN Suffix Adicional: Inclusão do sufixo @techcorp.com nas propriedades de Active Directory Domains and Trusts para preparar o mapeamento de logins corporativos.

🔒 Capítulo 3: Governança de Acesso e File Server (FS-01)
3.1 Arquitetura de Armazenamento
O serviço de arquivos foi isolado em um servidor membro exclusivo (FS-01), com adição de um segundo disco virtual VHDX de 10 GB formatado em NTFS sob a letra E:\ e rótulo DadosTechCorp.

3.2 O Padrão AGDLP
A concessão de privilégios de acesso aos diretórios foi implementada rigorosamente sob a metodologia AGDLP (Account -> Global Group -> Domain Local Group -> Permission):

Contas de Usuário (A): Ex: lucas.silva

Grupos Globais (G): Agrupamento por setor no AD DS (GG_Comercial, GG_Financeiro, GG_RH, GG_TI, GG_Diretoria).

Grupos Domain Local (DL): Agrupamento do recurso no AD DS (DL_FS_Comercial_RW, DL_FS_Financeiro_RW, etc.).

Permissões NTFS (P): O grupo Domain Local recebe permissão explícita de Modificação (Modify) na pasta de seu respectivo setor dentro do volume E:\Empresa.

3.3 Automação de Pastas e Isolamento NTFS
A criação das pastas, quebra de herança, remoção de grupos genéricos e aplicação do recurso Access-Based Enumeration (ABE) no compartilhamento Empresa$ foi realizada pelo script abaixo no FS-01:

# 1. Variáveis e Criação de Diretórios
```powershell
$BasePath = "E:\Empresa"
$Folders = @("RH", "TI", "Financeiro", "Comercial", "Diretoria")

if (-not (Test-Path $BasePath)) { New-Item -Path$BasePath -ItemType Directory -Force }

foreach ($folder in $Folders) {$folderPath = Join-Path -Path $BasePath -ChildPath$folder
    if (-not (Test-Path $folderPath)) { New-Item -Path$folderPath -ItemType Directory -Force }
}
```

# 2. Criação do Compartilhamento SMB com ABE
New-SmbShare -Name "Empresa$" -Path $BasePath -FullAccess "TECHCORP\Domain Admins" -ReadAccess "TECHCORP\Domain Users"
Set-SmbShare -Name "Empresa$" -FolderEnumerationMode AccessBased -Force

# 3. Quebra de Herança e Aplicação do AGDLP
```powershell
foreach ($folder in $Folders) {$targetPath = Join-Path -Path $BasePath -ChildPath$folder
    $groupName = "TECHCORP\DL_FS_${folder}_RW"

    $acl = Get-Acl -Path $targetPath$acl.SetAccessRuleProtection($true,$true) # Quebra herança e converte
    $acl.Access \vert{} Where-Object {$_.IdentityReference -like "*Users*" } | ForEach-Object { $acl.RemoveAccessRule($_) }

    $permission =$groupName, "Modify", "ContainerInherit, ObjectInherit", "None", "Allow"
    $accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule $permission
    $acl.AddAccessRule($accessRule)

    Set-Acl -Path $targetPath -AclObject$acl
}
```
📜 Capítulo 4: Objetos de Diretiva de Grupo (GPO)
A padronização das estações de trabalho e a segurança do ambiente foram impostas centralizadamente via GPMC (gpmc.msc) no DC01:

4.1 GPO-01 — Bloqueio de Mídia Removível (USB)
Escopo: Aplicada na OU Financeiro (e escalável para outras OUs restritas).

Configuração: Computer Configuration > Administrative Templates > System > Removable Storage Access.

Ação: Habilitada a negação de execução, leitura e escrita para discos removíveis (Removable Disks: Deny all access).

4.2 GPO-02 — Política Global de Senhas e Bloqueio de Conta
Escopo: Vinculada na raiz do domínio techcorp.local.

Configuração: Computer Configuration > Windows Settings > Security Settings > Account Policies.

Ação:

Tamanho mínimo da senha: 14 caracteres.

Complexidade de senha: Habilitada (letras maiúsculas, minúsculas, números e símbolos).

Duração máxima da senha: 90 dias.

Limiar de bloqueio de conta: 5 tentativas incorretas (bloqueio por 30 minutos).

4.3 GPO-03 — Mapeamento Dinâmico de Unidade de Rede
Escopo: Aplicada na OU Workstations / Users.

Configuração: User Configuration > Preferences > Windows Settings > Drive Maps.

Ação: Criação dinâmica da unidade S: com a ação Update:

Caminho do Recurso: \\FS-01\Empresa$\%Department%

Comportamento: O Windows resolve a variável %Department% do usuário logado e monta o drive S: apontando diretamente para a pasta do seu setor no FS-01. Devido ao ABE, o usuário visualiza apenas a sua pasta.

✅ Capítulo 5: Matriz de Validação e Testes On-Premises
Validação de Login e GPO (CLI-01): Realizado logon com o usuário lucas.silva (Setor Comercial). O comando gpresult /r confirmou a aplicação da GPO-02 e GPO-03.

Validação do Mapeamento Dinâmico: Abertura do File Explorer no Windows 11 exibindo a unidade S: criada automaticamente e apontando para \\FS-01\Empresa$\Comercial.

Validação do Isolamento de Segurança (NTFS / ABE):

Ao navegar no compartilhamento \\FS-01\Empresa$, o usuário visualiza apenas a pasta Comercial (demais pastas ocultas pelo ABE).

Tentativa de acesso direto via caminho UNC \\FS-01\Empresa$\Financeiro retornou a mensagem nativa "Acesso Negado", validando a quebra de herança e o isolamento AGDLP.