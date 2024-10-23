# Imersão TFTEC Azure ao vivo em SP

Esse projeto descreve os principais passos utilizados no nosso treinamento presencial de SP.
Temos como objetivo construir uma infraestrutura completa, simulando um cenário real de uma empresa que tem a necessidade de utilizar diversos recursos do Azure para rodar o seu negócio.

# Estrutura
O desenho de arquitetura informado abaixo mostra alguns detalhes de como está configurada a estrutura da empresa que iremos construir.

![TFTEC Cloud](https://github.com/raphasi/tftecaovivosp24/blob/main/Diagrama.png "Arquitetura Imersão")

## STEP01 - Criar um Resource Group e estrutura de VNETS e Subnets
1- Script PowerShell para criar estrutura de rede inicial
Realizar download do Script e importar no Azure CloudShell: https://github.com/raphasi/tftecaovivosp24/blob/main/Script_Landing_Zone.ps1
```cmd
## Script: Criar Landing Zone - TFTEC ao VIVO SP
## Autor: Raphael Andrade
## Data: Setembro/24

# Definir variáveis
$resourceGroupName = "rg-tftecsp-001"
$location01 = "uksouth"
$location02 = "eastus"

# Criar Resource Group
New-AzResourceGroup -Name $resourceGroupName -Location $location01

# Criar VNet Hub e Subnet
$subnetConfigHub = New-AzVirtualNetworkSubnetConfig -Name "sub-srv-001" -AddressPrefix "10.10.1.0/24"
$vnetHub = New-AzVirtualNetwork -ResourceGroupName $resourceGroupName -Location $location02 -Name "vnet-hub-001" -AddressPrefix "10.10.0.0/16" -Subnet $subnetConfigHub

# Criar NSG Hub e associar à subnet
$nsgHub = New-AzNetworkSecurityGroup -ResourceGroupName $resourceGroupName -Location $location02 -Name "nsg-hub-001"

# Adicionar regra de entrada para RDP
$nsgHub | Add-AzNetworkSecurityRuleConfig -Name "Allow-RDP-VM-APPs" `
    -Description "Libera acesso RDP para VM-APPs" `
    -Access Allow `
    -Protocol Tcp `
    -Direction Inbound `
    -Priority 200 `
    -SourceAddressPrefix * `
    -SourcePortRange * `
    -DestinationAddressPrefix "10.10.1.4" `
    -DestinationPortRange 3389

# Aplicar as mudanças ao NSG
$nsgHub | Set-AzNetworkSecurityGroup

Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnetHub -Name "sub-srv-001" -AddressPrefix "10.10.1.0/24" -NetworkSecurityGroup $nsgHub
$vnetHub | Set-AzVirtualNetwork

# Criar VNet Spoke e Subnets
$subnetConfigSpoke = @(
    New-AzVirtualNetworkSubnetConfig -Name "sub-web-001" -AddressPrefix "10.11.1.0/24"
    New-AzVirtualNetworkSubnetConfig -Name "sub-aks-001" -AddressPrefix "10.11.2.0/24"
    New-AzVirtualNetworkSubnetConfig -Name "sub-appgw-001" -AddressPrefix "10.11.3.0/24"
    New-AzVirtualNetworkSubnetConfig -Name "sub-vint-001" -AddressPrefix "10.11.4.0/24"
    New-AzVirtualNetworkSubnetConfig -Name "sub-pvte-001" -AddressPrefix "10.11.5.0/24"
)
$vnetSpoke = New-AzVirtualNetwork -ResourceGroupName $resourceGroupName -Location $location01 -Name "vnet-spk-001" -AddressPrefix "10.11.0.0/16" -Subnet $subnetConfigSpoke

# Criar NSG Spoke e associar às subnets
$nsgSpoke = New-AzNetworkSecurityGroup -ResourceGroupName $resourceGroupName -Location $location01 -Name "nsg-spk-001"
$subnetsToAssociate = @("sub-web-001", "sub-aks-001", "sub-vint-001", "sub-pvte-001")
foreach ($subnetName in $subnetsToAssociate) {
    Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnetSpoke -Name $subnetName -AddressPrefix ($vnetSpoke.Subnets | Where-Object {$_.Name -eq $subnetName}).AddressPrefix -NetworkSecurityGroup $nsgSpoke
}
$vnetSpoke | Set-AzVirtualNetwork

# Criar peering entre as VNets
Add-AzVirtualNetworkPeering -Name "HubToSpoke" -VirtualNetwork $vnetHub -RemoteVirtualNetworkId $vnetSpoke.Id -AllowForwardedTraffic
Add-AzVirtualNetworkPeering -Name "SpokeToHub" -VirtualNetwork $vnetSpoke -RemoteVirtualNetworkId $vnetHub.Id -AllowForwardedTraffic

# Criar VM Windows
$vmName = "vm-apps"
$vmSize = "Standard_B2s"
$adminUsername = "admin.tftec"
$adminPassword = ConvertTo-SecureString "Partiunuvem@2024" -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ($adminUsername, $adminPassword)

# Verificar a disponibilidade da imagem
$publisher = "MicrosoftWindowsServer"
$offer = "WindowsServer"
$skus = @("2022-Datacenter", "2022-datacenter-azure-edition")

$availableSku = $null
foreach ($sku in $skus) {
    $availableImage = Get-AzVMImage -Location $location02 -PublisherName $publisher -Offer $offer -Skus $sku -ErrorAction SilentlyContinue
    if ($availableImage) {
        $availableSku = $sku
        break
    }
}

if (-not $availableSku) {
    Write-Error "Não foi possível encontrar uma imagem do Windows Server 2022 disponível na região $location02"
    return
}

$publicIp = New-AzPublicIpAddress -Name "$vmName-pip" -ResourceGroupName $resourceGroupName -Location $location02 -AllocationMethod Static
$nic = New-AzNetworkInterface -Name "$vmName-nic" -ResourceGroupName $resourceGroupName -Location $location02 -SubnetId $vnetHub.Subnets[0].Id -PublicIpAddressId $publicIp.Id

# Definir a configuração da VM
$vmConfig = New-AzVMConfig -VMName $vmName -VMSize $vmSize

# Definir o sistema operacional
$vmConfig = Set-AzVMOperatingSystem -VM $vmConfig -Windows -ComputerName $vmName -Credential $credential -ProvisionVMAgent -EnableAutoUpdate

# Definir a imagem do sistema operacional
$vmConfig = Set-AzVMSourceImage -VM $vmConfig -PublisherName $publisher -Offer $offer -Skus $availableSku -Version "latest"

# Adicionar a interface de rede
$vmConfig = Add-AzVMNetworkInterface -VM $vmConfig -Id $nic.Id

# Configurar o disco do sistema operacional
$vmConfig = Set-AzVMOSDisk -VM $vmConfig -CreateOption FromImage -StorageAccountType Premium_LRS

# Criar a VM
New-AzVM -ResourceGroupName $resourceGroupName -Location $location02 -VM $vmConfig

# Criar Storage Account
$storageAccountName = "stotftecsp" + (Get-Random -Minimum 100000 -Maximum 999999)
$storageAccount = New-AzStorageAccount -ResourceGroupName $resourceGroupName `
                                       -Name $storageAccountName `
                                       -Location $location01 `
                                       -SkuName Standard_LRS `
                                       -Kind StorageV2

# Criar container 'imagens' na Storage Account
$ctx = $storageAccount.Context
New-AzStorageContainer -Name "imagens" -Context $ctx -Permission Off

Write-Output "Storage Account criada: $storageAccountName"
 ```
2- Aplicar Azure Policy
   - Aplicar Policy no nível do Resource Group criado no passo anterior
   - Aplicar policy para recursos herdarem a TAG do RG-AZURE (Inherit a tag from the resource group)
   - Aplicar policy para limitar o tamanho das VMs para B2S e D2S_V3 (Allowed virtual machine size SKUs)
   - Configurar mensagem de não complice: Você escolheu um SKU de VM não permitido, Utilize apenas B2S ou D2S_V3
   



## STEP02 - Deploy dos WebApps
1.0 Criar um App Service Plan:
```cmd
   Nome: appplan-tftec-001
   Operation System: Windows
   Região: uksouth
   SKU: Standard S1
```

1.1 Criar o WebApp do Ingresso
```cmd
   Nome: app-ingresso-tftecxxxx
   Desmarcar "Unique default hostname"
   Runtime Stack: .NET 8
   Operation Sistem: Windows
   Region: uksouth
   Escolher o App Service Plan criado no passo 1.0
```

1.2 Criar o WebApp do Bend (API)
```cmd
   Nome: app-bend-tftecxxxx
   Desmarcar "Unique default hostname"
   Runtime Stack: .NET 8
   Operation Sistem: Windows
   Region: uksouth
   Escolher o App Service Plan criado no passo 1.0
```

1.3 Criar o Webapp do CRM
```cmd 
   Nome: app-crm-tftecxxxx
   Desmarcar "Unique default hostname"
   Runtime Stack: .NET 8
   Operation Sistem: Windows
   Region: uksouth
   Escolher o App Service Plan criado no passo 1.0
 ```

 1.4 Criar o Webapp do Auth
```cmd 
   Nome: app-auth-tftecxxxx
   Desmarcar "Unique default hostname"
   Runtime Stack: .NET 8
   Operation Sistem: Windows
   Region: uksouth
   Escolher o App Service Plan criado no passo 1.0
 ```  
2.0 Baixar pacotes das aplicações:
https://github.com/raphasi/imersaoazure

2.1 Descompactar o .zip e utilizando o cloudshell, fazer upload dos 4 pacotes.

2.2 Realizar o deploy da aplicação BEND (API) para o WebApp
Abrir o Powershell ou Terminal e executar o seguinte comando:
```cmd
az login (ou utilizar o CloudShell)
az webapp deploy --resource-group rg-azure --name app-ingresso-tftec --src-path ingresso.zip
```

2.3 Realizar o deploy da aplicação INGRESSO para o WebApp
Abrir o Powershell ou Terminal e executar o seguinte comando:
```cmd
az login (ou utilizar o CloudShell)
az webapp deploy --resource-group rg-azure --name app-bend-tftec --src-path bend.zip
```

2.4 Realizar o deploy da aplicação CRM para o WebApp
Abrir o Powershell ou Terminal e executar o seguinte comando:
```cmd
az login (ou utilizar o CloudShell)
az webapp deploy --resource-group rg-azure --name app-crm --src-path crm.zip
```

2.5 Realizar o deploy da aplicação AUTH para o WebApp
Abrir o Powershell ou Terminal e executar o seguinte comando:
```cmd
az login (ou utilizar o CloudShell)
az webapp deploy --resource-group rg-azure --name app-auth --src-path app-auth.zip
```

## STEP03 - Deploy do Azure SQL Database
1.0 Criar um novo SQL Server
```cmd
Nome: srv-sql-tftecxxxxx (usar um nome único)
Location: 
Authentication method: Use SQL authentication   
   user: sqladmin
   pass: Partiunuvem@2024
Allow Azure services and resources to access this server: YES
```
1.1 Instalar o SSMS
```cmd
Acessar o servidor vm-apps e instalar o SQL Management Studio
```
Download do SQL SSMS: https://aka.ms/ssmsfullsetup

1.2 Importar database aplicação WebSite
```cmd
Abrir o SQL Management Studio
Server Name: Copia o nome do SQL Server criado no passo anterior
Alterar formato de autenticação para SQL Server authentication  
Logar com usuário e senha criados no passo anterior
Importar o database usando a opção de dacpac
*Caso necessário, alterar o nome do database para: sistema-tftec-db
```
1.3 Ajustar SQL Database
```cmd
Ajustar configuração do SQL Database:
   - Compute + Storage: Mudar opção de backup para LRS
```

## STEP04 - Ajuste Connection String
1.0 Configurar conexão BEND x SQL Database
```cmd
Realizar o ajuste da connection string no WebApp BEND
Testar o Swagger validando uma consulta no banco
```

## STEP05 - Deploy Logic App  
1.0 Realizar o deploy do Logic App para update das imagens do WebApp INGRESSO
```cmd
Logic App Tier: Consumption
Resource Group: rg-tftecsp-001
Logic App name: lgapp-img-001
Enable log analytics: NO
```
1.1 Configurar o fluxo para o import das imagens
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DOS FLUXOS
```


## STEP06 - Deploy Apps Registration para o CRM
1.0 Criar o App Registration CRM01
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DO APPREGISTRATION
```

1.1 Criar o App Registration CRM02
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DO APPREGISTRATION
```

## STEP07 - Configurar as variáveis de ambiente crm
1.0 Configurar as variáveis de ambiente da aplicação CRM
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DAS VARIÁVEIS DE AMBIENTE
```

1.1 Testar o acesso CRM autenticando com o Entra ID
```cmd
Acessar o endereço do webapp de CRM
Na tela de login do Entra ID, autenticar com um usuário com permissão
```

## STEP08 - Deploy do Tenant do Azure B2C
1.0 Criar um novo Azure Active Directory B2C
```cmd
Cllicar em "Create a resource"
Digitar a opção "Azure Active Directory B2C"
Escolher "Create a new Azure AD B2C Tenant"
Digitar um nome para a organização
Digitar um domain name para a organização
Escolher o Resource Group: rg-tftecsp
```

## STEP09 - Deploy Apps Registration demais aplicações
1.0 Criar o App Registration 03
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DO APPREGISTRATION
```
1.1 Criar o App Registration 04
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DO APPREGISTRATION
```

## STEP07 - Configurar as variáveis de ambiente demais ambientes
1.0 Configurar as variáveis de ambiente da aplicação INGRESSO
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DAS VARIÁVEIS DE AMBIENTE DA APLICAÇÃO INGRESSO
```
1.1 Configurar as variáveis de ambiente da aplicação BEND
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DAS VARIÁVEIS DE AMBIENTE DA APLICAÇÃO BEND
```
1.2 Configurar as variáveis de ambiente da aplicação AUTH
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DAS VARIÁVEIS DE AMBIENTE DA APLICAÇÃO AUTH
```

## STEP10 - Realizar teste completo para todas as aplicações
1.0 Testar experiência no ambiente de Compra e CRM
```cmd
BEND: Realizar um teste de GET nos dados de custumer
INGRESSO: Realizar o cadastro de um usuário no site utilizando AZURE B2C e realizar uma compra de produto.
CRM: Realizar a autenticação utilizando um usuário com permissão no Entra ID e validar a compra realizada no passo anterior.
```

## STEP11 - Realizar ajustes de conectividade no SQL Database e WebApps
1.0 Configurar o SQL Database para trabalhar com private endpoint
```cmd
Acessar o menu Networking
Public network access: Disable
Private Access - Criar um private endpoint
Name: pvt-endp-sql-001
Network Interface Name: pvt-endp-sql-001-nic
Region: uksouth
Target sub-resource: sqlServer
Virtual network: vnet-spk-001
Subnet: sub-db-001
Dynamically allocate IP address
```
1.1 Configurar VNET Integration para o WebApps INGRESSO
```cmd
Acessar o menu Networking
Clicar em Virtual network integration - Not configured
Add Virtual Network Integration
Virtual Network: vnet-spk-001
Subnet: sub-vint-001
```

1.2 Configurar VNET Integration para o WebApps CRM
```cmd
Acessar o menu Networking
Clicar em Virtual network integration - Not configured
Add Virtual Network Integration
Selecionar a connection já criada no passo anterior
```

1.3 Configurar VNET Integration para o WebApps BEND
```cmd
Acessar o menu Networking
Clicar em Virtual network integration - Not configured
Add Virtual Network Integration
Selecionar a connection já criada no passo anterior
```

1.4 Configurar VNET Integration para o WebApps AUTH
```cmd
Acessar o menu Networking
Clicar em Virtual network integration - Not configured
Add Virtual Network Integration
Selecionar a connection já criada no passo anterior
```

1.5 Realizar testes de acesso
```cmd
Teste via BEND - Swagger
Teste de login no CRM via Entra ID
Teste de login B2C no CRM
```


## STEP12 - Deploy certificados Digitais
1.0 Gerar um certificado digital válido:
https://punchsalad.com/ssl-certificate-generator/
```cmd
Criar um certificado digital para a URL que será utilizada em cada uma das aplicações (usando como sufixo o seu domínio público.)
```

1.1 Converter o certificado para PFX:
https://www.sslshopper.com/ssl-converter.html
```cmd
Gerar o certificado cno formato pfx
Cadastrar uma senha simples para o certificado. Exemplo: tftec2024
```

1.2 repetir o passo de criação 3 vezes:
```cmd
 - Certificado para aplicação INGRESSO
 - Certificado para aplicação BEND (api)
 - Certificado para aplicação CRM
```

## STEP13 - Deploy Azure Key Vault
1.0 Deploy Azure Key Vault:
```cmd
   Nome: kvault-tftec-001
   Região: uksouth
   Configurar o acesso ao Key Vault como Access Policy
```

1.1 Fazer upload dos certificado PFX no Key Vault
```cmd
Fazer upload do certificado pfx da aplicação INGRESSO
Fazer upload do certificado pfx da aplicação CRM
Fazer upload do certificado pfx da aplicação BEND (API)
```

## STEP14 - Criar um Managed Identity
1.0 Criar um Managed Identity para liberar acesso do AppGw aos certificados do KeyVault
```cmd
Resource Group: rg-tftecsp-001
Region: uksouth
Name: mgtid-kvault-certs
```
1.2 Liberando acesso do Managed Identity no Key Vault
```cmd
Acessar o Key Vault crido no STEP12
Adicionar uma Access policies
Secret e Certification permitions: GET
```

## STEP15 - Deploy do Application Gateway
1.0 Deploy Application Gateway e configuração do App Ingresso:
```cmd
Resource group: rg-tftecsp-prd
Name: appgw-web-001
Region: uksouth
Tier: Standard v2
Enable autoscaling: Yes
IP address type: IPV4 only
Virtual Network: vnet-spoke-001
Subnet: sub-appgw-001
Frontend IP address type: Public
Create Public IPV4: pip-appgw-001
Add a backend pool: bpool-ingresso (Associar ao WebApp de Ingresso)
Add a routing rule
Rule name: web-ingresso-https
Priority: 100
Listener name: lst-ing-https
Protocol: HTTPS
Choose a certificate from Key Vault
Cert name: cert-ingresso
Managed identity: mngid-kv-001
Certificate: cert-ingresso
Listener type: Multi site
Host name: ingresso.seudominiopublico
Target type: Backend pool
Backend target: bpool.ingresso
Backend settings: Add new
Backend settings name: sts-ingresso-https
Backend server’s certificate is issued by a well-known CA: YES
Override with new host name: YES
Host name: FQDN do seu WebApp de ingresso
```
1.1 Configuração do App CRM:
Backend pools
```cmd
Adicionar um backendpool
Name: bpool-crm
Target type: App Services (Associar ao WebApp de CRM)
```
Backend settings
```cmd
Backend settings name: sts-crm-https
Protocol: HTTPS
Override with new host name: YES
Host name: Default domain do seu WebApp de CRM
```
Health probes
```cmd
Name: proble-crm
Protocol: HTTPS
Host: Default domain do seu WebApp de CRM
Path: /
Backend settings: sts-crm-https
```
Listeners
```cmd
Listener name: lst-web-crm-https
Frontend IP: Public
Protocol: HTTPS
Choose a certificate: Create new
Selecionar o certificado referente a aplicação CRM
Listener type: Multi site
Hostname: crm.seudominiopublico
```
Rules
```cmd
Rule name: web-crm-https
Priority: 102
Listener: lst-web-crm-https
Backend targets
Target type: Backend pool
Backend target: bpool-crm
Backend settings:  sts-auth-https 
```

1.2 Configuração do App BEND (API):
Backend pools
```cmd
Adicionar um backendpool
Name: bpool-bend
Target type: App Services (Associar ao WebApp de BEND)
```
Backend settings
```cmd
Backend settings name: sts-bend-https
Protocol: HTTPS
Override with new host name: YES
Host name: Default domain do seu WebApp de BEND
```
Health probes
```cmd
Name: proble-bend
Protocol: HTTPS
Host: Default domain do seu WebApp de BEND
Path: /swagger
Backend settings: sts-bend-https
```
Listeners
```cmd
Listener name: lst-web-bend-https
Frontend IP: Public
Protocol: HTTPS
Choose a certificate: Create new
Selecionar o certificado referente a aplicação BEND
Listener type: Multi site
Hostname: api.seudominiopublico
```
Rules
```cmd
Rule name: web-bend-https
Priority: 103
Listener: lst-web-bend-https
Backend targets
Target type: Backend pool
Backend target: bpool-bend
Backend settings:  sts-bend-https 
```

## STEP16 - Ajustar URLs de autenticação
1.0 Ajustar as URLs de autenticação OIDC nos App Registrations
```cmd
Acessar o APP Registrartion e alterar a URL xxxxxx
Acessar o APP Registrartion e alterar a URL xxxxxx
Acessar o APP Registrartion e alterar a URL xxxxxx
Acessar o APP Registrartion e alterar a URL xxxxxx
```
1.1 Ajustar as URLs de autenticação OIDC nos WebApps
```cmd
Acessar o WebApp xxx e alterar a URL xxxxxx
Acessar o WebApp xxx e alterar a URL xxxxxx
```


## STEP17 - Configurar o Application Insights
1.0 Realizar o deploy do Log Analytics Workspaces
```cmd
Resource group: rg-tftecsp-001
Name: wksloganl001
Region: uksouth
```
1.1 Habilitar o Application Insights no WebApp Ingresso
```cmd
Habilitar o Application Insights direcionando os logs para o Workspace criado no passo 1.0
```
1.2 Habilitar o Application Insights no WebApp BEND
```cmd
Habilitar o Application Insights direcionando os logs para o Workspace criado no passo 1.0
```

## STEP18 - Deploy WebApp SLOTS
1.0 Criar um slot para o WebApp do INGRESSO
```cmd
Acessar o WebApp INGRESSO
Ir na opção Deployment slots - Add slot
Name: dev (ele irá complementar o nome com o default name do WebApp principal
Clone settings from: Do not clone settings
```
1.1 Criar um slot para o WebApp do CRM
```cmd
Acessar o WebApp CRM
Ir na opção Deployment slots - Add slot
Name: dev (ele irá complementar o nome com o default name do WebApp principal
Clone settings from: Do not clone settings
```
1.1 Criar um slot para o WebApp do BEND
```cmd
Acessar o WebApp BEND
Ir na opção Deployment slots - Add slot
Name: dev (ele irá complementar o nome com o default name do WebApp principal
Clone settings from: Do not clone settings
```

## STEP19 - Criar banco de DEV
1.0 Criar uma base no Azure SQL Database para DEV
```cmd
Acessar o o database de produção sistema-tftec-db
Selecionar a opção de copy
Database name sistema-tftec-db-dev
Server: o mesmo de produção
Want to use SQL elastic pool?: NO
Compute + storage: Basic
Backup storage redundancy: Locally-redundant backup storage
```

## STEP20 - Ajuste Connection String DEV e Private Connections
1.1 Configurar VNET Integration para o WebApps BEND-DEV
```cmd
Acessar o menu Networking
Clicar em Virtual network integration - Not configured
Add Virtual Network Integration
Virtual Network: vnet-spk-001
Subnet: sub-vint-001
```
1.2 Configurar VNET Integration para o WebApps INGRESSO-DEV
```cmd
Acessar o menu Networking
Clicar em Virtual network integration - Not configured
Add Virtual Network Integration
Virtual Network: vnet-spk-001
Subnet: sub-vint-001
```
1.2 Configurar VNET Integration para o WebApps CRM-DEV
```cmd
Acessar o menu Networking
Clicar em Virtual network integration - Not configured
Add Virtual Network Integration
Virtual Network: vnet-spk-001
Subnet: sub-vint-001
```

2.0 Configurar conexão BEND-DEV x SQL Database DEV
```cmd
Realizar o ajuste da connection string no WebApp BEND-DEV
Testar o Swagger validando uma consulta no banco
```

## STEP21 - Configurar as variáveis de ambiente para os SLOTs de DEV
1.0 Configurar as variáveis de ambiente da aplicação INGRESSO-DEV
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DAS VARIÁVEIS DE AMBIENTE DA APLICAÇÃO INGRESSO
```
1.1 Configurar as variáveis de ambiente da aplicação BEND-DEV
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DAS VARIÁVEIS DE AMBIENTE DA APLICAÇÃO BEND
```
1.1 Configurar as variáveis de ambiente da aplicação CRM-DEV
```cmd
DESCREVER OS PASSOS PARA CONFIGURAÇÃO DAS VARIÁVEIS DE AMBIENTE DA APLICAÇÃO BEND
```






# PASSOS UTILIZADOS NO EVENTO DE 2023 #



## STEP21 - Deploy APIM - API Management service
  ```cmd
   Cluster present configuration: Standard
   Nome: apim-tftec01
   Região: east-us
   Organization Name: TFTEC Cloud
   Administrator email: seu email para notificações
   Pricing tier: Developer
   ```
   
## STEP22 - Import Azure SQL Database
```cmd
Abrir o SQL Management Studio
Server Name: Copiar o nome do SQL Server já existente
Alterar formato de autenticação para SQL Server authentication  
Logar com usuário e senha usados na criação do banco
Importar o database usando a opção de dacpac
Manter o nome do database como apim_database
```

## STEP23 - Deploy WebApp API
1- Criar um WebApp
```cmd
   Nome: tftecapi01 (usar seu nome exclusivo)
   Publish: Code
   Runtime Stack: .NET6
   Região: east-us
   Escolher AppServicePlan já criado
```
2- Deploy da aplicação
Baixar o zip da aplicação em 
https://github.com/raphasi/imersaoazure

3- Realizar o deploy da aplicação para o WebApp
Abrir o Powershell ou Terminal e executar o seguinte comando:
```cmd
az login (ou utilizar o CloudShell)
az webapp deploy --resource-group rg-azure --name <app-name> --src-path DeploymentAPI.zip
```
4- Ajustar application setting para endereço do SQL Database
   - Acessar o WebApp - Configuration
   - Connection string
   - New connection string
   - Add/Edit connection string
  ```cmd
   Name: DefaultConnection
   Value: Server=tcp:sqlsrvtftec00001.database.windows.net,1433;Initial Catalog=apim_database;Persist Security Info=False;User ID=adminsql;Password=Partiunuvem@2023;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;
   Type: SQLAzure
   SAVE
   ``` 
   
5- Ajustar conexão interna do WebApp com o Azure SQL Database
   - Acessar o WebApp criado no passo anterior
   - Networking
   - Outbound Traffic
   - Vnet Integration
   - Add VNet
   - Escolher a vnet-hub
   - Escolher a subnet sub-db


## STEP24 - Deploy WebApp Gerenciador/Interface
1- Criar um WebApp
```cmd
   Nome: appinterface001 (usar seu nome exclusivo)
   Publish: Code
   Runtime Stack: .NET6
   Região: east-us
   Escolher AppServicePlan já criado
```
2- Deploy da aplicação
Baixar o zip da aplicação em 
https://github.com/raphasi/imersaoazure

3- Realizar o deploy da aplicação para o WebApp
Abrir o Powershell ou Terminal e executar o seguinte comando:
```cmd
az login (ou utilizar o CloudShell)
az webapp deploy --resource-group rg-azure --name <app-name> --src-path DeploymentGerenciador.zip
```
4- Ajustar application settings 
   - Acessar o WebApp - Configuration
   - Application settings
   - New application settings
   - Add/Edit application settings
  ```cmd
   Name: ServiceUri:UrlApi
   Value: https://apim-tftec00001.azure-api.net (coletar a URL do APIM)
   SAVE
   ``` 

## STEP25 - Expor requests no APIM
1- Disponibilizar APIs externamente no APIM
   - Acessar o APIM criado
   - Acessar APIs
   - Em Create from definition, escolher a opção OpenAPI
   - Clicar em Select a file e importar o arquivo APICatalogo.openapi+json (baixado do diretório do github, dentro da pasta APICatalog)
   - Clicar em Create
   - Acessar em Design, a seção Backend
   - Clicar em editar HTTP(s) endpoint
   - Marcar a opção override e adicionar o endereço do WebApp (APIM - tftecapi01)
   - Exemplo: https://tftecapi000001.azurewebsites.net/
   - SAVE
   - Acessar a aba Settings
   - Apagar o conteúdo do campo Web service URL
   - Desmarcar a opção Subscription required
   - SAVE
   




