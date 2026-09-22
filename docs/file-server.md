# 📂 Capítulo: Migração e Governança do File Server Dedicado (FS-01)

Este documento detalha o processo de arquitetura, provisionamento, automação e migração da estrutura de arquivos corporativos da **TechCorp**, transferindo o compartilhamento de rede do Domain Controller (`DC01`) para um servidor de arquivos dedicado (**`FS-01`**).

---

## 📌 Passo 1: Motivação e Arquitetura da Solução

### 1.1 Objetivo Técnico
Desacoplar a função de File Server do Domain Controller (`DC01`) para seguir as boas práticas de segurança e governança da Microsoft, garantindo:
* **Isolamento de Funções:** Evitar que serviços de arquivos comprometam a segurança e a performance dos serviços de domínio (AD DS/DNS/DHCP).
* **Escalabilidade de Armazenamento:** Dedicar um volume de dados exclusivo (`E:\DadosTechCorp`) gerenciado de forma independente do disco de sistema operacional (`C:\`).

### 1.2 Endereçamento e Topologia
* **Hostname:** `FS-01`
* **Endereço IP:** `192.168.10.12 /24`
* **Gateway:** `192.168.10.1`
* **DNS Principal:** `192.168.10.10` (`DC01`)
* **Volume de Dados:** Disco Virtual VHDX de 10 GB (`E:\`)

---

## ⚙️ Passo 2: Provisionamento e Preparação do FS-01

### 2.1 Adição do Disco de Dados e Formatação
1. Inclusão de um novo disco rígido virtual de expansão dinâmica (100 GB) via Hyper-V SCSI Controller.
2. Inicialização da tabela de partição como **GPT**.
3. Formatação do volume como **NTFS** sob o rótulo **`DadosTechCorp`** e atribuição da letra de unidade **`E:`**.

### 2.2 Integração ao Domínio
Comunicação DNS e ingresso do servidor no domínio ativo via PowerShell:
```powershell
# Alteração de nome e IP estático executados previamente
Add-Computer -DomainName "techcorp.local" -Credential (Get-Credential) -Restart
```
🤖 Passo 3: Automação da Estrutura e Permissões (AGDLP)
A criação do diretório raiz E:\Empresa, das subpastas departamentais, do compartilhamento SMB oculto e das listas de controle de acesso (ACLs) foi 100% automatizada via PowerShell.

3.1 Script de Automação Aplicado

```powershell
# 1. Parâmetros e Variáveis de Caminho
$DriveLetter = "E:"
$BasePath = "$DriveLetter\Empresa"
$Folders = @("RH", "TI", "Financeiro", "Comercial", "Diretoria")

# 2. Criar a estrutura de diretórios no disco de dados
if (-not (Test-Path $BasePath)) {
    New-Item -Path $BasePath -ItemType Directory -Force
}

foreach ($folder in $Folders) {$folderPath = Join-Path -Path $BasePath -ChildPath$folder
    if (-not (Test-Path $folderPath)) {
        New-Item -Path $folderPath -ItemType Directory -Force
    }
}

# 3. Criar o Compartilhamento SMB com ABE (Access-Based Enumeration)
New-SmbShare -Name "Empresa$" -Path $BasePath -FullAccess "TECHCORP\Domain Admins" -ReadAccess "TECHCORP\Domain Users"
Set-SmbShare -Name "Empresa$" -FolderEnumerationMode AccessBased -Force

# 4. Aplicar Isolamento NTFS e Modelo AGDLP
foreach ($folder in $Folders) {$targetPath = Join-Path -Path $BasePath -ChildPath$folder
    $groupName = "TECHCORP\DL_FS_${folder}_RW"

    # Quebra de herança NTFS e remoção de permissões genéricas
    $acl = Get-Acl -Path $targetPath$acl.SetAccessRuleProtection($true,$true)
    $acl.Access \vert{} Where-Object {$_.IdentityReference -like "*Users*" } | ForEach-Object { $acl.RemoveAccessRule($_) }

    # Concessão do acesso Modify ao grupo Domain Local correspondente
    $permission =$groupName, "Modify", "ContainerInherit, ObjectInherit", "None", "Allow"
    $accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule $permission
    $acl.AddAccessRule($accessRule)

    # Aplicação da nova ACL
    Set-Acl -Path $targetPath -AclObject$acl
}
```
🛡️ Passo 4: Modelo de Segurança e Governança de Acesso
4.1 Padrão AGDLP Implementado
Account (A): Usuário individual (ex: lucas.silva).

Global Group (G): Agrupamento por função/departamento (ex: GG_Comercial).

Domain Local Group (DL): Agrupamento que atribui a permissão no recurso (ex: DL_FS_Comercial_RW).

Permission (P): Acesso de Modificação (NTFS Modify) na pasta de destino E:\Empresa\Comercial.

4.2 Resource Isolation & Access-Based Enumeration (ABE)
Quebra de Herança: As pastas departamentais não herdam permissões do diretório pai (E:\Empresa).

Negação Implícita: Grupos genéricos (Users / Domain Users) foram removidos das ACLs NTFS das subpastas.

ABE Ativado: Usuários só conseguem visualizar a pasta do seu próprio setor no compartilhamento de rede. Pastas para as quais o usuário não possui permissão de leitura são ocultadas automaticamente pelo Windows.

🔄 Passo 5: Redirecionamento de Mapeamento via GPO
Para garantir a transparência da migração para os usuários finais, a GPO-03 (Mapeamento Dinâmico de Rede) foi atualizada no DC01:

Caminho Antigo: \\DC01\Empresa$\%Department%

Novo Caminho Dedicado: \\FS-01\Empresa$\%Department%

Mapeamento: Unidade S: criada automaticamente no login do usuário de acordo com a variável de ambiente do departamento no AD.

✅ Passo 6: Testes e Validação da Migração
Acesso Dinâmico (CLI-01): Logon com o usuário lucas.silva (Setor Comercial) validando a montagem automática do Drive S:.

Validação ABE: Verificação de que apenas a pasta Comercial é visível/acessível dentro do compartilhamento \\FS-01\Empresa$.

Teste de Bloqueio NTFS: Tentativa de acesso direto via caminho \\FS-01\Empresa$\Financeiro resultando em Acesso Negado, confirmando a eficácia da quebra de herança e do modelo AGDLP.