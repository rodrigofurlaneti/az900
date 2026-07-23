# Parte 1: Consultas Prévias (Preparação)
## Passo 1: Consultar o Top 5 regiões mais baratas globalmente (Versão Bash)

Linux
```bash
curl -s "https://prices.azure.com/api/retail/prices?currencyCode='USD'&\$filter=armSkuName%20eq%20'Standard_B2s'%20and%20priceType%20eq%20'Consumption'" | jq -r '.Items[] | select(.productName | contains("Windows") | not) | select(.productName | contains("Spot") | not) | "\(.armRegionName) \(.retailPrice)"' | sort -k2 -n | head -n 5
```

Windows
```bash
$uri = "https://prices.azure.com/api/retail/prices?currencyCode='USD'&`$filter=armSkuName%20eq%20'Standard_B2s'%20and%20priceType%20eq%20'Consumption'"
(Invoke-RestMethod -Uri $uri).Items | 
    Where-Object { $_.productName -notmatch "Windows" -and $_.productName -notmatch "Spot" } | 
    Sort-Object retailPrice | 
    Select-Object armRegionName, retailPrice -First 5 | 
    Format-Table
```

## Passo 2: Consultar quais regiões estão liberadas na SUA assinatura
```bash
az vm list-skus --size Standard_B2s --all --query "[?name=='Standard_B2s' && length(restrictions) == \`0\`].locations[0]" -o table
```

# Parte 2: Criação da Infraestrutura (Execução)
# Definindo Variáveis para não ter que digitar repetido (Altere se necessário)
```bash
REGIAO="denmarkeast"
RG="RG-App-Novo"
VM_NOME="VirtualMachine"
VNET="vnet-denmark-b"
SUBNET="snet-denmark-b"
```

# Passo 3: Criar o grupo de recurso
```bash
az group create --name $RG --location $REGIAO
```

# Passo 4: Criar a Rede Virtual e a Sub-rede
```bash
az network vnet create \
    --resource-group $RG \
    --name $VNET \
    --location $REGIAO \
    --address-prefix 172.16.0.0/16 \
    --subnet-name $SUBNET \
    --subnet-prefix 172.16.0.0/24
```

# Passo 5: Criar a Máquina Virtual (Standard_B2s) e sem IP Público inicialmente
```bash
az vm create \
    --resource-group $RG \
    --name $VM_NOME \
    --location $REGIAO \
    --size Standard_B2s \
    --image Ubuntu2204 \
    --admin-username azureuser \
    --generate-ssh-keys \
    --vnet-name $VNET \
    --subnet $SUBNET \
    --public-ip-address ""
```

# Passo 6: Configurar o Desligamento Automático para Economizar (ex: 16:00 UTC)
```bash
az vm auto-shutdown \
    --resource-group $RG \
    --name $VM_NOME \
    --time 1600 \
    --location $REGIAO
```

# Passo 7: Criar o IP Público
```bash
az network public-ip create \
    --resource-group $RG \
    --name ${VM_NOME}-PublicIP \
    --location $REGIAO \
    --allocation-method Static \
    --sku Basic
```

# Passo 8: Associar o IP Público à Placa de Rede da sua VM
```bash
az network nic ip-config update \
    --resource-group $RG \
    --nic-name ${VM_NOME}VMNic \
    --name ipconfig1 \
    --public-ip-address ${VM_NOME}-PublicIP
```

# Passo 9: Liberar as Portas de 80 a 85 no Firewall (NSG)
```bash
az network nsg rule create \
    --resource-group $RG \
    --nsg-name ${VM_NOME}NSG \
    --name Allow_APIs_80_85 \
    --protocol Tcp \
    --priority 100 \
    --destination-port-ranges 80-85 \
    --source-address-prefixes '*' \
    --access Allow \
    --direction Inbound
```
