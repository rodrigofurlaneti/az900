# Passo 1: Criar o grupo de recurso
```bash
az group create --name ResourceGroup --location eastus
```

# Passo 2: Criar a Rede Virtual e a Sub-rede
```bash
az network vnet create \
    --resource-group ResourceGroup \
    --name vnet-eastus-b \
    --location eastus \
    --address-prefix 172.16.0.0/16 \
    --subnet-name snet-eastus-b \
    --subnet-prefix 172.16.0.0/24
```

# Passo 3: Criar a Máquina Virtual com a Série Econômica (Standard_B2s) e sem IP Público
```bash
az vm create \
    --resource-group ResourceGroup \
    --name VirtualMachine \
    --location eastus \
    --size Standard_B2s \
    --image Ubuntu2204 \
    --admin-username azureuser \
    --generate-ssh-keys \
    --vnet-name vnet-eastus-b \
    --subnet snet-eastus-b \
    --public-ip-address ""
```

# Passo 4: Configurar o Desligamento Automático para Economizar (ex: 16:00)
```bash
az vm auto-shutdown \
    --resource-group ResourceGroup \
    --name VirtualMachine \
    --time 1600 \
    --location eastus
```

# Passo 5: Criar o IP Público
```bash
az network public-ip create \
    --resource-group ResourceGroup \
    --name VirtualMachine-PublicIP \
    --location eastus \
    --allocation-method Static \
    --sku Basic
```

# Passo 6: Associar o IP Público à Placa de Rede da sua VM
```bash
az network nic ip-config update \
    --resource-group ResourceGroup \
    --nic-name VirtualMachineVMNic \
    --name ipconfig1 \
    --public-ip-address VirtualMachine-PublicIP
```

# Passo 7: Liberar as Portas de 80 a 85 no Firewall (NSG)
```bash
az network nsg rule create \
    --resource-group ResourceGroup \
    --nsg-name VirtualMachineNSG \
    --name Allow_APIs_80_85 \
    --protocol Tcp \
    --priority 100 \
    --destination-port-ranges 80-85 \
    --source-address-prefixes '*' \
    --access Allow \
    --direction Inbound
```

