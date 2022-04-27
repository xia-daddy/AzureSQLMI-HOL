# AzureSQL Managed Instance

기존 환경 마이그레이션 혹은 신규 서비스 구축 시 진행해야 할 기본 구축 가이드 및 일부 기능들을 HandsOn 합니다.  

## 1.변수설정

SQL Manged Instance 인스턴스를 만들려면 Azure 내에서 여러 리소스를 만들어야 하므로 Azure PowerShell 명령은 변수를 사용하여 환경을 간소화합니다. 변수를 정의한 다음, 동일한 PowerShell 세션 내의 각 섹션에서 cmdlet을 실행합니다.
```powershell
$NSnetworkModels = "Microsoft.Azure.Commands.Network.Models"
$NScollections = "System.Collections.Generic"
# The SubscriptionId in which to create these objects
$SubscriptionId = 'b7fa084d-0d77-4846-9fe6-f2a983192b73'   #구독ID 입력
# Set the resource group name and location for your managed instance
$resourceGroupName = "myResourceGroup-$(Get-Random)"
$location = "eastus"
# Set the networking values for your managed instance
$vNetName = "myVnet-$(Get-Random)"
$vNetAddressPrefix = "10.0.0.0/16"
$miSubnetName = "myMISubnet-$(Get-Random)"
$miSubnetAddressPrefix = "10.0.0.0/24"
#Set the managed instance name for the new managed instance
$instanceName = "myMIName-$(Get-Random)"
# Set the admin login and password for your managed instance
$miAdminSqlLogin = "msadmin"   #관리자ID
$miAdminSqlPassword = "P@ssw0rd1765"   #관리자PW
# Set the managed instance service tier, compute level, and license mode
$edition = "General Purpose"
$vCores = 4
$maxStorage = 128
$computeGeneration = "Gen5"
$license = "LicenseIncluded" #"BasePrice" or LicenseIncluded if you have don't have SQL Server licence that can be used for AHB discount
```

## 2.리소스그룹구성

먼저 Azure에 연결하고, 구독 컨텍스트를 설정하고, 리소스 그룹을 만듭니다.

이렇게 하려면 다음 PowerShell 스크립트를 실행합니다.
```powershell
## Connect to Azure
Connect-AzAccount

# Set subscription context
Set-AzContext -SubscriptionId $SubscriptionId 

# Create a resource group
$resourceGroup = New-AzResourceGroup -Name $resourceGroupName -Location $location -Tag @{Owner="SQLDB-Samples"}
```

## 3.네트워킹구성

리소스 그룹을 만든 후 가상 네트워크, 서브넷, 네트워크 보안 그룹 및 라우팅 테이블과 같은 네트워킹 리소스를 구성합니다.

이렇게 하려면 다음 PowerShell 스크립트를 실행합니다.
```powershell
# Configure virtual network, subnets, network security group, and routing table
$virtualNetwork = New-AzVirtualNetwork `
                      -ResourceGroupName $resourceGroupName `
                      -Location $location `
                      -Name $vNetName `
                      -AddressPrefix $vNetAddressPrefix

                  Add-AzVirtualNetworkSubnetConfig `
                      -Name $miSubnetName `
                      -VirtualNetwork $virtualNetwork `
                      -AddressPrefix $miSubnetAddressPrefix |
                  Set-AzVirtualNetwork
                  
$scriptUrlBase = 'https://raw.githubusercontent.com/Microsoft/sql-server-samples/master/samples/manage/azure-sql-db-managed-instance/delegate-subnet'

$parameters = @{
    subscriptionId = $SubscriptionId
    resourceGroupName = $resourceGroupName
    virtualNetworkName = $vNetName
    subnetName = $miSubnetName
    }

Invoke-Command -ScriptBlock ([Scriptblock]::Create((iwr ($scriptUrlBase+'/delegateSubnet.ps1?t='+ [DateTime]::Now.Ticks)).Content)) -ArgumentList $parameters

$virtualNetwork = Get-AzVirtualNetwork -Name $vNetName -ResourceGroupName $resourceGroupName
$miSubnet = Get-AzVirtualNetworkSubnetConfig -Name $miSubnetName -VirtualNetwork $virtualNetwork
$miSubnetConfigId = $miSubnet.Id
```

## 4.인스턴스생성

보안 강화를 위해 SQL Managed Instance 자격 증명에 대해 복잡한 임의 암호를 만듭니다.
```powershell
# Create credentials
$secpassword = ConvertTo-SecureString $miAdminSqlPassword -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ($miAdminSqlLogin, $secpassword)
```
그런 다음, SQL Managed Instance를 만듭니다.
```powershell
# Create managed instance
New-AzSqlInstance -Name $instanceName `
                      -ResourceGroupName $resourceGroupName -Location $location -SubnetId $miSubnetConfigId `
                      -AdministratorCredential $credential `
                      -StorageSizeInGB $maxStorage -VCore $vCores -Edition $edition `
                      -ComputeGeneration $computeGeneration -LicenseType $license
```

SQL MI 생성은 장시간이 걸리는 작업입니다.
대략 4~5시간 정도 소요 후 생성이 완료됩니다.


## 5. 




