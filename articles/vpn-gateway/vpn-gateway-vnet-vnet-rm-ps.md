<properties
   pageTitle="Azure VNets verbinden met VPN Gateway en PowerShell | Microsoft Azure"
   description="Dit artikel helpt u bij het met elkaar verbinden van virtuele netwerken met behulp van Azure Resource Manager en PowerShell."
   services="vpn-gateway"
   documentationCenter="na"
   authors="cherylmc"
   manager="carmonm"
   editor=""
   tags="azure-resource-manager"/>

<tags
   ms.service="vpn-gateway"
   ms.devlang="na"
   ms.topic="get-started-article"
   ms.tgt_pltfrm="na"
   ms.workload="infrastructure-services"
   ms.date="08/31/2016"
   ms.author="cherylmc"/>


# Met PowerShell een VNet-naar-VNet-verbinding configureren voor Resource Manager

> [AZURE.SELECTOR]
- [Resource Manager - PowerShell](vpn-gateway-vnet-vnet-rm-ps.md)
- [Klassiek - Klassieke portal](virtual-networks-configure-vnet-to-vnet-connection.md)

Dit artikel helpt u met de stappen voor het maken van een verbinding tussen VNets in het Resource Manager-implementatiemodel met behulp van VPN Gateway. De virtuele netwerken kunnen zich in dezelfde of verschillende regio's bevinden en tot dezelfde of verschillende abonnementen behoren.


![v2v-diagram](./media/vpn-gateway-vnet-vnet-rm-ps/v2vrmps.png)


### Implementatiemodellen en -methoden voor VNet-naar-VNet


[AZURE.INCLUDE [vpn-gateway-clasic-rm](../../includes/vpn-gateway-classic-rm-include.md)] 

Een VNet-naar-VNet-verbinding kan worden geconfigureerd in beide implementatiemodellen en met verschillende hulpprogramma's. Deze volgende tabel wordt bijgewerkt naarmate nieuwe artikelen en aanvullende hulpprogramma's voor deze configuratie beschikbaar komen. Wanneer een artikel beschikbaar is, koppelen we dit rechtstreeks aan de tabel.<br><br>

[AZURE.INCLUDE [vpn-gateway-table-vnet-vnet](../../includes/vpn-gateway-table-vnet-to-vnet-include.md)] 

#### VNet-peering

[AZURE.INCLUDE [vpn-gateway-vnetpeeringlink](../../includes/vpn-gateway-vnetpeeringlink-include.md)]


## Over VNet-naar-VNet-verbindingen

Het verbinden van een virtueel netwerk met een ander virtueel netwerk (VNet-naar-VNet) lijkt op het verbinden van een VNet met een on-premises locatie. Voor beide connectiviteitstypen wordt een Azure VPN-gateway gebruikt om een beveiligde tunnel met IPsec/IKE te bieden. De VNets die u verbindt, kunnen zich in verschillende regio's bevinden. Of tot verschillende abonnementen behoren. U kunt zelfs VNet-naar-VNet-communicatie met multi-site-configuraties combineren. Zoals u in het volgende diagram kunt zien, kunt u netwerktopologieën maken waarin cross-premises connectiviteit wordt gecombineerd met connectiviteit tussen virtuele netwerken:


![Over verbindingen](./media/vpn-gateway-vnet-vnet-rm-ps/aboutconnections.png)
 
### Waarom virtuele netwerken koppelen?

U wilt virtuele netwerken wellicht koppelen om de volgende redenen:

- **Geografische redundantie en aanwezigheid tussen regio's**
    - U kunt uw eigen geo-replicatie of synchronisatie met beveiligde connectiviteit instellen zonder gebruik te maken van internetgerichte eindpunten.
    - Met Azure Traffic Manager en Load Balancer kunt u workloads met maximale beschikbaarheid instellen met behulp van geografische redundantie over meerdere Azure-regio's. Een belangrijk voorbeeld hiervan is het instellen van SQL Always On met beschikbaarheidsgroepen verspreid over meerdere Azure-regio's.

- **Regionale toepassingen met meerdere lagen met isolatie- of beheergrenzen**
    - Binnen dezelfde regio kunt u vanwege isolatie- of beheervereisten toepassingen met meerdere lagen instellen met meerdere virtuele netwerken die met elkaar zijn verbonden.


### Veelgestelde vragen over VNet-naar-VNet

[AZURE.INCLUDE [vpn-gateway-vnet-vnet-faq](../../includes/vpn-gateway-vnet-vnet-faq-include.md)] 


## Welke stappen moet ik gebruiken?

In dit artikel ziet u twee verschillende reeksen stappen. Een reeks stappen voor [VNets die zich in hetzelfde abonnement bevinden](#samesub) en een andere voor [VNets die zich in verschillende abonnementen bevinden](#difsub). Het belangrijkste verschil tussen de reeksen is of u alle resources van het virtuele netwerk en de gateway kunt maken en configureren binnen dezelfde PowerShell-sessie.

In de stappen in dit artikel worden variabelen gebruikt die aan het begin van elke sectie zijn gedeclareerd. Als u al met bestaande VNets werkt, wijzigt u de variabelen zo dat ze overeenkomen met de instellingen in uw eigen omgeving. 

![Beide verbindingen](./media/vpn-gateway-vnet-vnet-rm-ps/differentsubscription.png)


## <a name="samesub"></a>VNets verbinden die tot hetzelfde abonnement behoren

![v2v-diagram](./media/vpn-gateway-vnet-vnet-rm-ps/v2vrmps.png)

### Voordat u begint
    
Voordat u begint, moet u de Azure Resource Manager PowerShell-cmdlets installeren. Zie [How to install and configure Azure PowerShell](../powershell-install-configure.md) (Azure PowerShell installeren en configureren) voor meer informatie over het installeren van de PowerShell-cmdlets.

### <a name="Step1"></a>Stap 1: De IP-adresbereiken plannen


In de volgende stappen maakt u twee virtuele netwerken en hun bijbehorende gatewaysubnetten en configuraties. Vervolgens maakt u een VPN-verbinding tussen de twee VNets. Het is belangrijk dat u de IP-adresbereiken voor uw netwerkconfiguratie plant. De VNet-bereiken of de bereiken van het lokale netwerk mogen elkaar niet overlappen.

In de voorbeelden worden de volgende waarden gebruikt:

**Waarden voor TestVNet1:**

- VNet-naam: TestVNet1
- Resourcegroep: TestRG1
- Locatie: VS - oost
- TestVNet1: 10.11.0.0/16 en 10.12.0.0/16
- FrontEnd: 10.11.0.0/24
- BackEnd: 10.12.0.0/24
- GatewaySubnet: 10.12.255.0/27
- DNS-server: 8.8.8.8
- GatewayName: VNet1GW
- Openbare IP: VNet1GWIP
- VPNType: op route gebaseerd
- Connection(1to4): VNet1toVNet4
- Connection(1to5): VNet1toVNet5
- ConnectionType: VNet2VNet


**Waarden voor TestVNet4:**

- VNet-naam: TestVNet4
- TestVNet2: 10.41.0.0/16 & 10.42.0.0/16
- FrontEnd: 10.41.0.0/24
- BackEnd: 10.42.0.0/24
- GatewaySubnet: 10.42.255.0/27
- Resourcegroep: TestRG4
- Locatie: VS - west
- DNS-server: 8.8.8.8
- GatewayName: VNet4GW
- Openbare IP: VNet4GWIP
- VPNType: op route gebaseerd
- Verbinding: VNet4toVNet1
- ConnectionType: VNet2VNet



### <a name="Step2"></a>Stap 2: TestVNet1 maken en configureren

1. De variabelen declareren

    Declareer eerst de variabelen. In dit voorbeeld worden de variabelen gedeclareerd met de waarden voor deze oefening. In de meeste gevallen moet u de waarden vervangen door uw eigen waarden. U kunt deze variabelen echter gebruiken als u de stappen alleen wilt doorlopen om vertrouwd te raken met dit type configuratie. Wijzig de variabelen als dat nodig is en kopieer en plak ze vervolgens in uw PowerShell-console.

        $Sub1 = "Replace_With_Your_Subcription_Name"
        $RG1 = "TestRG1"
        $Location1 = "East US"
        $VNetName1 = "TestVNet1"
        $FESubName1 = "FrontEnd"
        $BESubName1 = "Backend"
        $GWSubName1 = "GatewaySubnet"
        $VNetPrefix11 = "10.11.0.0/16"
        $VNetPrefix12 = "10.12.0.0/16"
        $FESubPrefix1 = "10.11.0.0/24"
        $BESubPrefix1 = "10.12.0.0/24"
        $GWSubPrefix1 = "10.12.255.0/27"
        $DNS1 = "8.8.8.8"
        $GWName1 = "VNet1GW"
        $GWIPName1 = "VNet1GWIP"
        $GWIPconfName1 = "gwipconf1"
        $Connection14 = "VNet1toVNet4"
        $Connection15 = "VNet1toVNet5"

2. Verbinding maken met uw abonnement

    Schakel over naar de PowerShell-modus om de Resource Manager-cmdlets te gebruiken. Open de PowerShell-console en maak verbinding met uw account. Gebruik het volgende voorbeeld als hulp bij het maken van de verbinding:

        Login-AzureRmAccount

    Controleer de abonnementen voor het account.

        Get-AzureRmSubscription 

    Geef het abonnement op dat u wilt gebruiken.

        Select-AzureRmSubscription -SubscriptionName $Sub1

3. Een nieuwe resourcegroep maken

        New-AzureRmResourceGroup -Name $RG1 -Location $Location1

4. De subnetconfiguraties maken voor TestVNet1

    In dit voorbeeld wordt een virtueel netwerk gemaakt met de naam TestVNet1. Er worden ook drie subnetten gemaakt, GatewaySubnet, FrontEnd en BackEnd. Wanneer u de waarden vervangt, is het belangrijk dat u de juiste namen voor de gatewaysubnets gebruikt, in het bijzonder GatewaySubnet. Als u een andere naam kiest, mislukt het maken van de gateway. 

    In het volgende voorbeeld worden de variabelen gebruikt die u eerder hebt ingesteld. In dit voorbeeld maakt het gatewaysubnet gebruik van een /27. Hoewel u een gatewaysubnet ook kunt maken met een subnet van slechts /29, wordt dit niet aanbevolen. Een iets groter subnet wordt aanbevolen, bijvoorbeeld een /27 of /26. Op die manier kunt u bestaande of toekomstige configuraties benutten waarvoor mogelijk een groter gatewaysubnet is vereist. 

        $fesub1 = New-AzureRmVirtualNetworkSubnetConfig -Name $FESubName1 -AddressPrefix $FESubPrefix1
        $besub1 = New-AzureRmVirtualNetworkSubnetConfig -Name $BESubName1 -AddressPrefix $BESubPrefix1
        $gwsub1 = New-AzureRmVirtualNetworkSubnetConfig -Name $GWSubName1 -AddressPrefix $GWSubPrefix1


5. TestVNet1 maken

        New-AzureRmVirtualNetwork -Name $VNetName1 -ResourceGroupName $RG1 `
        -Location $Location1 -AddressPrefix $VNetPrefix11,$VNetPrefix12 -Subnet $fesub1,$besub1,$gwsub1

6. Een openbaar IP-adres aanvragen

    Vraag om een openbaar IP-adres toe te wijzen aan de gateway die u voor uw VNet gaat maken. De toewijzingsmethode is Dynamisch. U kunt het IP-adres dat u wilt gebruiken niet zelf opgeven. Het wordt dynamisch toegewezen aan uw gateway. 

        $gwpip1 = New-AzureRmPublicIpAddress -Name $GWIPName1 -ResourceGroupName $RG1 `
        -Location $Location1 -AllocationMethod Dynamic

7. De gatewayconfiguratie maken

    De gatewayconfiguratie bepaalt welk subnet en openbaar IP-adres moeten worden gebruikt. Gebruik het voorbeeld om de gatewayconfiguratie te maken. 

        $vnet1 = Get-AzureRmVirtualNetwork -Name $VNetName1 -ResourceGroupName $RG1
        $subnet1 = Get-AzureRmVirtualNetworkSubnetConfig -Name "GatewaySubnet" -VirtualNetwork $vnet1
        $gwipconf1 = New-AzureRmVirtualNetworkGatewayIpConfig -Name $GWIPconfName1 `
        -Subnet $subnet1 -PublicIpAddress $gwpip1

8. De gateway voor TestVNet1 maken

    In deze stap maakt u de virtuele netwerkgateway TestVNet1. VNet-naar-VNet-configuraties vereisen een op route gebaseerd VpnType. Het maken van een gateway kan even duren (45 minuten of langer).

        New-AzureRmVirtualNetworkGateway -Name $GWName1 -ResourceGroupName $RG1 `
        -Location $Location1 -IpConfigurations $gwipconf1 -GatewayType Vpn `
        -VpnType RouteBased -GatewaySku Standard

### Stap 3: TestVNet4 maken en configureren

Wanneer u TestVNet1 hebt geconfigureerd, maakt u TestVNet4. Volg de stappen hieronder en vervang de waarden door uw eigen waarden wanneer dat nodig is. Deze stap kan worden uitgevoerd binnen dezelfde PowerShell-sessie omdat deze tot hetzelfde abonnement behoort.

1. De variabelen declareren

    Zorg ervoor dat u de waarden vervangt door de waarden die u voor uw configuratie wilt gebruiken.

        $RG4 = "TestRG4"
        $Location4 = "West US"
        $VnetName4 = "TestVNet4"
        $FESubName4 = "FrontEnd"
        $BESubName4 = "Backend"
        $GWSubName4 = "GatewaySubnet"
        $VnetPrefix41 = "10.41.0.0/16"
        $VnetPrefix42 = "10.42.0.0/16"
        $FESubPrefix4 = "10.41.0.0/24"
        $BESubPrefix4 = "10.42.0.0/24"
        $GWSubPrefix4 = "10.42.255.0/27"
        $DNS4 = "8.8.8.8"
        $GWName4 = "VNet4GW"
        $GWIPName4 = "VNet4GWIP"
        $GWIPconfName4 = "gwipconf4"
        $Connection41 = "VNet4toVNet1"

    Controleer voordat u verder gaat of u nog bent verbonden met abonnement 1.

2. Een nieuwe resourcegroep maken

        New-AzureRmResourceGroup -Name $RG4 -Location $Location4

3. De subnetconfiguraties maken voor TestVNet4

        $fesub4 = New-AzureRmVirtualNetworkSubnetConfig -Name $FESubName4 -AddressPrefix $FESubPrefix4
        $besub4 = New-AzureRmVirtualNetworkSubnetConfig -Name $BESubName4 -AddressPrefix $BESubPrefix4
        $gwsub4 = New-AzureRmVirtualNetworkSubnetConfig -Name $GWSubName4 -AddressPrefix $GWSubPrefix4

4. TestVNet4 maken

        New-AzureRmVirtualNetwork -Name $VnetName4 -ResourceGroupName $RG4 `
        -Location $Location4 -AddressPrefix $VnetPrefix41,$VnetPrefix42 -Subnet $fesub4,$besub4,$gwsub4

5. Een openbaar IP-adres aanvragen

        $gwpip4 = New-AzureRmPublicIpAddress -Name $GWIPName4 -ResourceGroupName $RG4 `
        -Location $Location4 -AllocationMethod Dynamic

6. De gatewayconfiguratie maken

        $vnet4 = Get-AzureRmVirtualNetwork -Name $VnetName4 -ResourceGroupName $RG4
        $subnet4 = Get-AzureRmVirtualNetworkSubnetConfig -Name "GatewaySubnet" -VirtualNetwork $vnet4
        $gwipconf4 = New-AzureRmVirtualNetworkGatewayIpConfig -Name $GWIPconfName4 -Subnet $subnet4 -PublicIpAddress $gwpip4

7. De TestVNet4-gateway maken

        New-AzureRmVirtualNetworkGateway -Name $GWName4 -ResourceGroupName $RG4 `
        -Location $Location4 -IpConfigurations $gwipconf4 -GatewayType Vpn `
        -VpnType RouteBased -GatewaySku Standard

### Stap 4: De gateways verbinden

1. Beide gateways van het virtuele netwerk verkrijgen

    Omdat beide gateways in dit voorbeeld tot hetzelfde abonnement behoren, kan deze stap worden uitgevoerd in dezelfde PowerShell-sessie.

        $vnet1gw = Get-AzureRmVirtualNetworkGateway -Name $GWName1 -ResourceGroupName $RG1
        $vnet4gw = Get-AzureRmVirtualNetworkGateway -Name $GWName4 -ResourceGroupName $RG4


2. De verbinding tussen TestVNet1 en TestVNet4 maken

    In deze stap maakt u de verbinding van TestVNet1 naar TestVNet4. Hier ziet u een gedeelde sleutel waarnaar wordt verwezen in de voorbeelden. U kunt uw eigen waarden voor de gedeelde sleutel gebruiken. Het belangrijkste is dat de gedeelde sleutel voor beide verbindingen moet overeenkomen. Het kan even duren voordat de verbinding is gemaakt.

        New-AzureRmVirtualNetworkGatewayConnection -Name $Connection14 -ResourceGroupName $RG1 `
        -VirtualNetworkGateway1 $vnet1gw -VirtualNetworkGateway2 $vnet4gw -Location $Location1 `
        -ConnectionType Vnet2Vnet -SharedKey 'AzureA1b2C3'

3. De verbinding tussen TestVNet4 en TestVNet1 maken

    Deze stap is vergelijkbaar met die hierboven, alleen maakt u de verbinding nu vanuit TestVNet4 naar TestVNet1. Zorg dat de gedeelde sleutels overeenkomen.

        New-AzureRmVirtualNetworkGatewayConnection -Name $Connection41 -ResourceGroupName $RG4 `
        -VirtualNetworkGateway1 $vnet4gw -VirtualNetworkGateway2 $vnet1gw -Location $Location4 `
        -ConnectionType Vnet2Vnet -SharedKey 'AzureA1b2C3'

    De verbinding moet na enkele minuten tot stand zijn gebracht.

4. Controleer de verbinding. Raadpleeg de sectie [De verbinding controleren](#verify).


## <a name="difsub"></a>VNets verbinden die tot verschillende abonnement behoren


![v2v-diagram](./media/vpn-gateway-vnet-vnet-rm-ps/v2vdiffsub.png)

In dit scenario worden TestVNet1 en TestVNet5 met elkaar verbonden. TestVNet1 en TestVNet5 bevinden zich in een verschillend abonnement. Met de stappen voor deze configuratie voegt u nog een VNet-naar-VNet-verbinding toe om TestVNet1 te verbinden met TestVNet5. 

Het verschil is hier dat een deel van de configuratiestappen moet worden uitgevoerd in een aparte PowerShell-sessie in de context van het tweede abonnement. Dit geldt met name wanneer de twee abonnementen tot verschillende organisaties behoren. 

De instructies volgen op de eerder beschreven stappen. U moet [Stap 1](#Step1) en [Stap 2](#Step2) uitvoeren om TestVNet1 en de VPN-gateway voor TestVNet1 te maken en te configureren. Nadat u stap 1 en stap 2 hebt voltooid, gaat u verder met stap 5 om TestVNet5 te maken.

### Stap 5: De extra IP-adresbereiken controleren

Het is belangrijk dat u controleert of de IP-adresruimte van het nieuwe virtuele netwerk, TestVNet5, niet overlapt met een van de VNet-bereiken of de gatewaybereiken van het lokale netwerk. 

In dit voorbeeld kunnen de virtuele netwerken tot verschillende organisaties behoren. Voor deze oefening kunt u de volgende waarden voor TestVNet5 gebruiken:

**Waarden voor TestVNet5:**

- VNet-naam: TestVNet5
- Resourcegroep: TestRG5
- Locatie: Japan - oost
- TestVNet5: 10.51.0.0/16 & 10.52.0.0/16
- FrontEnd: 10.51.0.0/24
- BackEnd: 10.52.0.0/24
- GatewaySubnet: 10.52.255.0.0/27
- DNS-server: 8.8.8.8
- GatewayName: VNet5GW
- Openbare IP: VNet5GWIP
- VPNType: op route gebaseerd
- Verbinding: VNet5toVNet1
- ConnectionType: VNet2VNet

**Aanvullende waarden voor TestVNet1:**

- Verbinding: VNet1toVNet5


### Stap 6: TestVNet5 maken en configureren

Deze stap moet worden uitgevoerd in de context van het nieuwe abonnement. Dit deel kan worden uitgevoerd door de beheerder in een andere organisatie die eigenaar is van het abonnement.

1. De variabelen declareren

    Zorg ervoor dat u de waarden vervangt door de waarden die u voor uw configuratie wilt gebruiken.

        $Sub5 = "Replace_With_the_New_Subcription_Name"
        $RG5 = "TestRG5"
        $Location5 = "Japan East"
        $VnetName5 = "TestVNet5"
        $FESubName5 = "FrontEnd"
        $BESubName5 = "Backend"
        $GWSubName5 = "GatewaySubnet"
        $VnetPrefix51 = "10.51.0.0/16"
        $VnetPrefix52 = "10.52.0.0/16"
        $FESubPrefix5 = "10.51.0.0/24"
        $BESubPrefix5 = "10.52.0.0/24"
        $GWSubPrefix5 = "10.52.255.0/27"
        $DNS5 = "8.8.8.8"
        $GWName5 = "VNet5GW"
        $GWIPName5 = "VNet5GWIP"
        $GWIPconfName5 = "gwipconf5"
        $Connection51 = "VNet5toVNet1"

2. Verbinding maken met abonnement 5

    Open de PowerShell-console en maak verbinding met uw account. Gebruik het volgende voorbeeld als hulp bij het maken van de verbinding:

        Login-AzureRmAccount

    Controleer de abonnementen voor het account.

        Get-AzureRmSubscription 

    Geef het abonnement op dat u wilt gebruiken.

        Select-AzureRmSubscription -SubscriptionName $Sub5

3. Een nieuwe resourcegroep maken

        New-AzureRmResourceGroup -Name $RG5 -Location $Location5

4. De subnetconfiguraties maken voor TestVNet4
    
        $fesub5 = New-AzureRmVirtualNetworkSubnetConfig -Name $FESubName5 -AddressPrefix $FESubPrefix5
        $besub5 = New-AzureRmVirtualNetworkSubnetConfig -Name $BESubName5 -AddressPrefix $BESubPrefix5
        $gwsub5 = New-AzureRmVirtualNetworkSubnetConfig -Name $GWSubName5 -AddressPrefix $GWSubPrefix5

5. TestVNet5 maken

        New-AzureRmVirtualNetwork -Name $VnetName5 -ResourceGroupName $RG5 -Location $Location5 `
        -AddressPrefix $VnetPrefix51,$VnetPrefix52 -Subnet $fesub5,$besub5,$gwsub5

6. Een openbaar IP-adres aanvragen

        $gwpip5 = New-AzureRmPublicIpAddress -Name $GWIPName5 -ResourceGroupName $RG5 `
        -Location $Location5 -AllocationMethod Dynamic


7. De gatewayconfiguratie maken

        $vnet5 = Get-AzureRmVirtualNetwork -Name $VnetName5 -ResourceGroupName $RG5
        $subnet5  = Get-AzureRmVirtualNetworkSubnetConfig -Name "GatewaySubnet" -VirtualNetwork $vnet5
        $gwipconf5 = New-AzureRmVirtualNetworkGatewayIpConfig -Name $GWIPconfName5 -Subnet $subnet5 -PublicIpAddress $gwpip5

8. De TestVNet5-gateway maken

        New-AzureRmVirtualNetworkGateway -Name $GWName5 -ResourceGroupName $RG5 -Location $Location5 `
        -IpConfigurations $gwipconf5 -GatewayType Vpn -VpnType RouteBased -GatewaySku Standard

### Stap 7: De gateways verbinden

Omdat de gateways in dit voorbeeld tot verschillende abonnementen behoren, is deze stap opgesplitst in twee PowerShell-sessies, aangeduid als [abonnement 1] en [abonnement 5].

1. **[Abonnement 1]** De gateway van virtueel netwerk verkrijgen voor abonnement 1

    Zorg dat u zich aanmeldt bij en verbinding maakt met abonnement 1.

        $vnet1gw = Get-AzureRmVirtualNetworkGateway -Name $GWName1 -ResourceGroupName $RG1

    Kopieer de uitvoer van de volgende elementen en stuur deze via e-mail of een andere manier naar de beheerder van abonnement 5.

        $vnet1gw.Name
        $vnet1gw.Id

    De waarden van deze twee elementen lijken op die in het volgende voorbeeld:

        PS D:\> $vnet1gw.Name
        VNet1GW
        PS D:\> $vnet1gw.Id
        /subscriptions/b636ca99-6f88-4df4-a7c3-2f8dc4545509/resourceGroupsTestRG1/providers/Microsoft.Network/virtualNetworkGateways/VNet1GW

2. **[Abonnement 5]** De gateway van virtueel netwerk verkrijgen voor abonnement 5

    Zorg dat u zich aanmeldt bij en verbinding maakt met abonnement 5.

        $vnet5gw = Get-AzureRmVirtualNetworkGateway -Name $GWName5 -ResourceGroupName $RG5

    Kopieer de uitvoer van de volgende elementen en stuur deze via e-mail of een andere manier naar de beheerder van abonnement 1.

        $vnet5gw.Name
        $vnet5gw.Id

    De waarden van deze twee elementen lijken op die in het volgende voorbeeld:

        PS C:\> $vnet5gw.Name
        VNet5GW
        PS C:\> $vnet5gw.Id
        /subscriptions/66c8e4f1-ecd6-47ed-9de7-7e530de23994/resourceGroups/TestRG5/providers/Microsoft.Network/virtualNetworkGateways/VNet5GW

3. **[Abonnement 1]** De verbinding tussen TestVNet1 en TestVNet5 maken

    In deze stap maakt u de verbinding van TestVNet1 naar TestVNet5. Het verschil is hier dat $vnet5gw niet rechtstreeks kan worden verkregen omdat het zich in een ander abonnement bevindt. U moet een nieuw PowerShell-object maken met de waarden die in de bovenstaande stappen zijn gecommuniceerd vanuit Abonnement 1. Gebruik onderstaand voorbeeld. Vervang de naam, de id en de gedeelde sleutel door uw eigen waarden. Het belangrijkste is dat de gedeelde sleutel voor beide verbindingen moet overeenkomen. Het kan even duren voordat de verbinding is gemaakt.

    Zorg dat u verbinding maakt met abonnement 1. 
    
        $vnet5gw = New-Object Microsoft.Azure.Commands.Network.Models.PSVirtualNetworkGateway
        $vnet5gw.Name = "VNet5GW"
        $vnet5gw.Id   = "/subscriptions/66c8e4f1-ecd6-47ed-9de7-7e530de23994/resourceGroups/TestRG5/providers/Microsoft.Network/virtualNetworkGateways/VNet5GW"
        $Connection15 = "VNet1toVNet5"
        New-AzureRmVirtualNetworkGatewayConnection -Name $Connection15 -ResourceGroupName $RG1 -VirtualNetworkGateway1 $vnet1gw -VirtualNetworkGateway2 $vnet5gw -Location $Location1 -ConnectionType Vnet2Vnet -SharedKey 'AzureA1b2C3'

4. **[Abonnement 5]** De verbinding tussen TestVNet5 en TestVNet1 maken

    Deze stap is vergelijkbaar met die hierboven, alleen maakt u de verbinding nu vanuit TestVNet5 naar TestVNet1. Hier moet op dezelfde manier een PowerShell-object worden gemaakt op basis van de waarden die zijn verkregen van abonnement 1. Zorg in deze stap dat de gedeelde sleutels overeenkomen.

    Zorg dat u verbinding maakt met abonnement 5.

        $vnet1gw = New-Object Microsoft.Azure.Commands.Network.Models.PSVirtualNetworkGateway
        $vnet1gw.Name = "VNet1GW"
        $vnet1gw.Id = "/subscriptions/b636ca99-6f88-4df4-a7c3-2f8dc4545509/resourceGroups/TestRG1/providers/Microsoft.Network/virtualNetworkGateways/VNet1GW "
        New-AzureRmVirtualNetworkGatewayConnection -Name $Connection51 -ResourceGroupName $RG5 -VirtualNetworkGateway1 $vnet5gw -VirtualNetworkGateway2 $vnet1gw -Location $Location5 -ConnectionType Vnet2Vnet -SharedKey 'AzureA1b2C3'

## <a name="verify"></a>Een verbinding controleren

[AZURE.INCLUDE [vpn-gateway-verify-connection-rm](../../includes/vpn-gateway-verify-connection-rm-include.md)]


## Volgende stappen

- Wanneer de verbinding is voltooid, kunt u virtuele machines aan uw virtuele netwerken toevoegen. Zie [Een virtuele machine maken](../virtual-machines/virtual-machines-windows-hero-tutorial.md) voor de stappen.
- Voor meer informatie over BGP raadpleegt u [BGP Overview](vpn-gateway-bgp-overview.md) (BGP-overzicht) en [How to configure BGP](vpn-gateway-bgp-resource-manager-ps.md) (BGP configureren). 




<!--HONumber=Oct16_HO1-->


