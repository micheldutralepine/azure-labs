# VM Disk Encryption via Azure CLI

Laboratório prático: criação de uma VM Windows Server 2022 (Spot), 
anexação de um disco de dados adicional, e criptografia de ambos os 
discos usando Azure Disk Encryption (ADE) integrado a um Key Vault.

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
