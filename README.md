# VM Disk Encryption via Azure CLI

Laboratório prático: criação de uma VM Windows Server 2022 (Spot), anexação de um disco de dados adicional, e criptografia de ambos os discos usando Azure Disk Encryption (ADE) integrado a um Key Vault.

## Arquitetura

- Resource Group: `rg-estudos`
- Região: `australiaeast`
- VM: `Standard_D2s_v5`, prioridade Spot, com VNet/Subnet customizadas
- Disco de dados adicional anexado e inicializado
- Key Vault com RBAC habilitado, guardando a chave de criptografia (KEK RSA 4096)
- Azure Disk Encryption aplicado em OS disk + Data disk (BitLocker)

## O que este lab cobre

- Autenticação e troca de tenant/subscription via `az login`
- Resolução de erros de cota (`QuotaExceeded`) vs disponibilidade de SKU (`SkuNotAvailable`)
- Criação de VM Spot com política de eviction
- Anexação de disco de dados via CLI
- Registro de resource provider (`Microsoft.KeyVault`)
- RBAC no Key Vault (`Key Vault Administrator`)
- Criação de chave de criptografia (KEK)
- Habilitação do Azure Disk Encryption via `az vm encryption enable`
- Verificação de criptografia em dois níveis: extensão da VM e configuração do disco

## Como rodar

Pré-requisitos: Azure CLI instalado, subscription ativa.

Antes de rodar, edite o arquivo `main.azcli` e substitua o valor da variável `pass` por uma senha sua.

```bash
az login
bash main.azcli
```

O script segue a ordem em que os comandos foram executados no lab. As etapas destrutivas (desativar criptografia e apagar o resource group) ficam comentadas por padrão, para não remover os recursos sem querer.

## Evidência

Confirmação de que a criptografia foi aplicada com sucesso nos dois discos (OS disk e Data disk), via `az vm encryption show`:
<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/a42f2901-f849-4584-8a5b-b0b3a0688a6f" />


## Problemas encontrados

Ao longo do lab apareceram 10 erros diferentes. Os mais relevantes:

- **Cota vs disponibilidade de SKU**: `QuotaExceeded` (cota zerada na subscription) e `SkuNotAvailable` (falta de capacidade na região) são erros parecidos mas com causas e soluções diferentes.
- **RBAC no Key Vault**: criar o Key Vault não dá automaticamente permissão para gerenciar chaves, é necessário atribuir uma role RBAC (`Key Vault Administrator`) explicitamente sobre o recurso.
- **Bug de ambiente no Git Bash/MINGW64**: o terminal converte automaticamente argumentos que começam com `/` em caminhos do Windows, corrompendo Resource IDs do Azure passados como parâmetro.

Lista completa com os 10 erros em [troubleshooting.md](./troubleshooting.md).

## Limpeza

```bash
az group delete -n rg-estudos --yes
```
