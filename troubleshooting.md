# Troubleshooting

Erros encontrados durante a execução deste lab, com a causa e a solução aplicada.

## 1. Alias de imagem incorreto

**Comando:**
```bash
az vm create -g $rg -n $vm --image windows2022datacenter ...
```

**Erro:**
```
(ResourceNotFound) The Resource 'Microsoft.Compute/images/windows2022datacenter'
under resource group 'rg-estudos' was not found.
```

**Causa:** o alias correto do CLI é `Win2022Datacenter` (com essa capitalização específica), não `windows2022datacenter`. Sem o alias certo, o CLI tenta procurar uma imagem custom com esse nome dentro do resource group, em vez de resolver para o Marketplace.

**Solução:**
```bash
az vm create -g $rg -n $vm --image Win2022Datacenter ...
```

---

## 2. Cota zerada no tamanho padrão de VM

**Erro:**
```
(QuotaExceeded) Operation could not be completed as it results in exceeding
approved standardDSv5Family Cores quota. Current Limit: 0, Current Usage: 0,
Additional Required: 2
```

**Causa:** quando `--size` não é especificado, o `az vm create` usa como padrão um tamanho da família `DSv5`. Nessa subscription, essa família tinha cota 0 (`az vm list-usage --location australiaeast --output table` confirmou o limite).

**Solução:** verificar quais famílias têm cota disponível e especificar um `--size` explícito de uma família com cota livre (nesse caso, `Standard DSv3 Family`, com limite 10).

---

## 3. SKU sem capacidade disponível na região (diferente de cota)

**Comando:**
```bash
az vm create -g $rg -n $vm --image $img --size Standard_B2s ...
```

**Erro:**
```
(SkuNotAvailable) The requested VM size for resource 'Following SKUs have
failed for Capacity Restrictions: Standard_B2s' is currently not available
in location 'australiaeast'.
```

**Causa:** `SkuNotAvailable` é diferente de `QuotaExceeded`. Não é falta de cota na subscription, é falta de capacidade física do datacenter para aquele tamanho específico naquele momento. Confirmado rodando:
```bash
az vm list-skus --location australiaeast --size Standard_B --output table --all
```
que mostrou `Standard_B2s` com `Restrictions: NotAvailableForSubscription`, enquanto `Standard_B2s_v2` (geração mais nova) aparecia com `Restrictions: None`.

**Solução:** usar a geração v2 do tamanho, ou outra família disponível.

---

## 4. Cota zerada em família diferente da testada

**Comando:**
```bash
az vm create -g $rg -n $vm --image $img --size Standard_B2ats_v2 ...
```

**Erro:**
```
(QuotaExceeded) ... standardBasv2Family Cores quota. Current Limit: 0
```

**Causa:** `Standard_B2ats_v2` pertence à família `standardBasv2Family`, que também tinha cota 0, diferente da família `Standard BS Family` (v1) que tinha cota 10. Nomes de família parecidos não implicam a mesma cota.

**Solução:** voltar para `Standard_D2s_v3`, da família `Standard DSv3`, confirmada com cota de 10 vCPUs livres.

---

## 5. VM Spot ignora a cota regular da família

Ao criar a VM com `--priority Spot`, o tamanho `Standard_D2s_v5` (família `DSv5`, cota 0) funcionou normalmente, mesmo essa família estando zerada para VMs regulares. Isso acontece porque instâncias Spot consomem a cota separada `Total Regional Low-priority vCPUs`, que tinha limite 3 disponível nessa subscription.

**Aprendizado:** cota de VM Spot é rastreada separadamente da cota regular por família.

---

## 6. Nome de disco duplicado ao anexar disco de dados

**Comando:**
```bash
az vm disk attach -g $rg --vm-name $vm --name $disk --new --size-gb 4
```

**Erro:**
```
(ConflictingUserInput) Disk disk-vmencrypt already exists in resource group
RG-ESTUDOS. Only CreateOption.Attach is supported.
```

**Causa:** a variável `$disk` (`disk-vmencrypt`) já estava em uso como nome do OS disk, definido no `--os-disk-name` da criação da VM. Não é possível criar um novo disco com o mesmo nome de um disco já existente.

**Solução:** usar um nome diferente para o disco de dados (`encryptdik2`).

---

## 7. Resource provider do Key Vault não registrado

**Erro:**
```
(MissingSubscriptionRegistration) The subscription is not registered to use
namespace 'Microsoft.KeyVault'.
```

**Causa:** subscription nova/pouco usada, que nunca havia utilizado o serviço Key Vault antes. Resource providers do Azure precisam ser registrados na subscription antes do primeiro uso.

**Solução:**
```bash
az provider register --namespace Microsoft.KeyVault
az provider show --namespace Microsoft.KeyVault --query registrationState
```
Aguardar o status mudar de `Registering` para `Registered` antes de tentar criar o Key Vault novamente.

---

## 8. Forbidden ao criar chave no Key Vault (RBAC)

**Erro:**
```
(Forbidden) Caller is not authorized to perform action on resource.
Action: 'Microsoft.KeyVault/vaults/keys/create/action'
Assignment: (not found)
Inner error: { "code": "ForbiddenByRbac" }
```

**Causa:** o Key Vault foi criado com `enableRbacAuthorization: true`. Nesse modelo, permissões não vêm do modelo antigo de Access Policies, e sim de roles RBAC atribuídas explicitamente sobre o recurso. Ser o criador do Key Vault não concede automaticamente permissão para gerenciar chaves.

**Solução:** atribuir a role `Key Vault Administrator` para o usuário sobre o escopo do Key Vault:
```bash
myobjectid=$(az ad signed-in-user show --query id -o tsv)
kvid=$(az keyvault show -n $kv -g $rg --query id -o tsv)
az role assignment create --role "Key Vault Administrator" --assignee $myobjectid --scope $kvid
```
A propagação da role leva de alguns segundos a poucos minutos.

---

## 9. `MissingSubscription` causado pelo Git Bash/MINGW64

**Erro:**
```
(MissingSubscription) The request did not have a subscription or a valid
tenant level resource provider.
```

**Causa:** o comando `az role assignment create` estava sintaticamente correto e as variáveis (`$myobjectid`, `$kvid`) estavam preenchidas corretamente (confirmado com `echo`). O problema era o ambiente: o Git Bash no Windows (MINGW64) converte automaticamente argumentos que começam com `/` para caminhos de arquivo do Windows, corrompendo o valor do `--scope` (que é um Resource ID do Azure, não um caminho de arquivo) antes de chegar ao Azure CLI.

**Solução:** desabilitar essa conversão automática para o comando específico:
```bash
MSYS_NO_PATHCONV=1 az role assignment create --role "Key Vault Administrator" --assignee $myobjectid --scope $kvid
```

---

## 10. `KeyError` ao consultar status de criptografia após desabilitar

**Comando:**
```bash
az vm encryption disable -g $rg -n $vm --volume-type ALL
az vm encryption show -g $rg -n $vm
```

**Erro:**
```
KeyError: 'encryptionSettings'
```

**Causa:** bug conhecido do Azure CLI. Depois que a criptografia é desabilitada, a extensão retorna um payload sem o campo `encryptionSettings`, mas o parser do comando `show` espera esse campo e falha ao processá-lo.

**Observação:** o `disable` em si concluiu com sucesso (sem erro). O erro ocorre apenas ao tentar consultar o status logo em seguida. Não impede o restante do fluxo (como o `az group delete`).
