## 1. Lab Title  
**Provisioning a Secure Web-Ready VM in Azure with Automated Configuration and Monitoring**

## 2. Overview & Objectives  

This lab teaches how to provision and secure an Azure VM (Linux or Windows), automate its configuration with web server setup scripts, and enable diagnostics using Azure Monitor.

- **User Story 1:** Provision a Linux VM to host a basic web service.  
  **Objective:** Use Azure PowerShell to create a Linux VM with associated networking.

- **User Story 2:** Configure NSG rules to allow only SSH (22) and HTTPS (443) traffic.  
  **Objective:** Define and apply Network Security Group rules via PowerShell.

- **User Story 3:** Script the installation and configuration of IIS/Nginx at VM startup.  
  **Objective:** Attach a custom data (cloud-init) or Custom Script Extension for consistent web server deployment.

- **User Story 4:** Enable Azure Monitor diagnostics on the VM.  
  **Objective:** Configure diagnostics settings through PowerShell to collect metrics and logs.

## 3. Prerequisites  

- Azure subscription (free tier or sandbox)  
- Azure CLI or Azure PowerShell module installed (`Install-Module -Name Az`)
  - _Tip:_ Run `az version` or `Get-Module -ListAvailable Az` to verify.
- Permissions to create resource groups, VMs, NSGs, and diagnostic settings  
- Access to Ubuntu and Windows Server 2019 images in the Azure region  
- Internet-connected local machine with CLI (Bash or PowerShell)

## 4. Architecture Diagram  


## 5. Step‑by‑Step Instructions  

### Step 1: Create Resource Group  
**Action:**  
```powershell
az group create --name LabResourceGrp --location eastus
```

**Explanation:**
Groups related Azure resources under one logical container for easy management.

**Validation:**
```powershell
Get-AzResourceGroup -Name LabResourceGrp | Format-Table
```

### Step 2: Create Network and NSG
**Action:**
```powershell
# Create VNet and Subnet
$vnet = New-AzVirtualNetwork `
  -Name LabVNet `
  -ResourceGroupName LabResourceGrp `
  -Location EastUS `
  -AddressPrefix "10.0.0.0/16"
Add-AzVirtualNetworkSubnetConfig `
  -VirtualNetwork $vnet `
  -Name LabSubnet `
  -AddressPrefix "10.0.0.0/24"
$vnet | Set-AzVirtualNetwork

# Create NSG
$nsg = New-AzNetworkSecurityGroup `
  -ResourceGroupName LabResourceGrp `
  -Location EastUS `
  -Name LabNSG

# Define and add rule for SSH & HTTPS
$rule = New-AzNetworkSecurityRuleConfig `
  -Name Allow-SSH-HTTPS `
  -Description "Allow SSH and HTTPS inbound" `
  -Access Allow `
  -Protocol Tcp `
  -Direction Inbound `
  -Priority 1000 `
  -SourceAddressPrefix * `
  -SourcePortRange * `
  -DestinationAddressPrefix * `
  -DestinationPortRange "22","80","443"

$nsg.SecurityRules.Add($rule)
$nsg | Set-AzNetworkSecurityGroup

```

**Explanation:**
Creates a virtual network with a subnet, then an NSG containing a rule permitting only ports 22 and 443.

**Validation:**
```powershell
Get-AzNetworkSecurityGroup -ResourceGroupName LabResourceGrp -Name LabNSG |
  Select-Object -ExpandProperty SecurityRules |
  Format-Table Name,Direction,Protocol,DestinationPortRange,Access,Priority
```

### Step 3: Create Public IP and Network Interface
**Action:**
```powershell
# Public IP
$pip = New-AzPublicIpAddress `
  -Name LabPublicIP `
  -ResourceGroupName LabResourceGrp `
  -Location EastUS `
  -AllocationMethod Static

# Retrieve subnet and NSG objects
$subnet = (Get-AzVirtualNetwork -Name LabVNet -ResourceGroupName LabResourceGrp).Subnets |
  Where-Object Name -EQ "LabSubnet"
$nsg    = Get-AzNetworkSecurityGroup -Name LabNSG -ResourceGroupName LabResourceGrp

# Network Interface
$nic = New-AzNetworkInterface `
  -Name LabNIC `
  -ResourceGroupName LabResourceGrp `
  -Location EastUS `
  -SubnetId $subnet.Id `
  -PublicIpAddressId $pip.Id `
  -NetworkSecurityGroupId $nsg.Id
```

**Explanation:**
Assigns a dynamic public IP and attaches the NIC to the VNet subnet and NSG for inbound security.

**Validation:**
```powershell
Get-AzNetworkInterface -Name LabNIC -ResourceGroupName LabResourceGrp |
  Select-Object Name,IpConfigurations | Format-List
```

### Step 4: Create Linux VM with Nginx Script
**Action:**
```bash
# variables
RG="LabResourceGrp"
LOC="eastus"
VNET="LabVNet"
SUBNET="LabSubnet"
NSG="LabNSG"
PIP="LabPublicIP"
NIC="LabNIC"
VM="LinuxWebVM"
IMG="Canonical:UbuntuServer:18.04-LTS:latest"
SSH_KEY_PATH="$HOME/.ssh/id_rsa.pub"

#Create cloud-init-nginx.yml file
cat <<EOF > cloud-init-nginx.yml
#cloud-config
packages:
  - nginx
runcmd:
  - systemctl enable nginx
  - systemctl start nginx
EOF

# Create VM
az vm create \
  --resource-group LabResourceGrp \
  --name LinuxWebVM \
  --location eastus \
  --nics LabNIC \
  --image Ubuntu2204 \
  --size Standard_DS1_v2 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init-nginx.yml
```

**Explanation:**
Builds a Linux VM, injects cloud-init to install/start Nginx for consistent web service deployment.

**Validation:**
```powershell
( Get-AzPublicIpAddress -Name LabPublicIP -ResourceGroupName LabResourceGrp ).IpAddress
# Then in browser: https://ipaddress
```

### Step 5: Enable Azure Monitor Diagnostics
**Action:**
```powershell
# Create Log Analytics workspace
$workspace = New-AzOperationalInsightsWorkspace `
  -ResourceGroupName LabResourceGrp `
  -Name LabLogs `
  -Location EastUS `
  -Sku Standard

# Get VM resource ID
$vm = Get-AzVM -ResourceGroupName LabResourceGrp -Name LinuxWebVM

# Configure diagnostics
Set-AzDiagnosticSetting `
  -ResourceId $vm.Id `
  -WorkspaceId $workspace.ResourceId `
  -Name VMMonitoring `
  -MetricCategory AllMetrics `
  -Enabled $true `
  -Category Syslog `
  -Enabled $true
```

**Explanation:**
Creates a Log Analytics workspace and pushes VM metrics/logs to Azure Monitor for ongoing analysis.

**Validation:**
Open Azure Portal → Monitor → Logs → run:
```kusto 
AzureMetrics | where ResourceId contains "LinuxWebVM"
```

## 6. Troubleshooting & FAQs
### **Error 1:** Cannot connect via SSH
- **Cause:** NSG rule missing or misconfigured.
- **Fix:** Verify NSG inbound rule and NIC association:
```powershell
Get-AzNetworkSecurityGroup -Name LabNSG |
  Select-Object -ExpandProperty SecurityRules
```

### **Error 2:** Nginx not running after VM creation
- **Cause:** cloud-init script failed.
- **Fix:** Inspect logs inside VM:
```bash
sudo journalctl -u cloud-init
sudo cat /var/log/cloud-init-output.log
```

### **Error 3:** Diagnostic setting deployment error
- **Cause:** Region mismatch or workspace name collision.
- **Fix:** Confirm workspace location matches VM region, and names are unique.

## 7. Expert Insights
- **Least Privilege:** Use role-based access (e.g., ```Contributor``` scoped to the resource group).

- **Immutable Infrastructure:** Bake common software into a custom VM image (Azure Image Builder) for faster deployment.

- **Monitoring:** Add boot diagnostics ```(-EnableBootDiagnostics)``` for VM startup troubleshooting.

- **Performance:** Choose Premium SSD disks and right-size VM SKU based on load tests.

- **Pro Tip:** Use Azure Automation Desired State Configuration (DSC) to enforce post-deployment configurations.

## 8. Knowledge-Retention Activities
### Reflection Questions
1. How does ```Set-AzDiagnosticSetting``` integrate with Log Analytics to surface VM metrics?

2. Why might you use a custom image over cloud-init for production deployments?

3. In what scenarios would you choose NSG over Azure Firewall?


### Quiz
1. **What ports are allowed in this lab's NSG?**

- A. 22 and 80
- B. 22 and 443 ✅
- C. 80 and 443
- D. 3389 and 443

1. **What parameter specifies custom data (cloud-init) in New-AzVm?**

- A. -CustomData ✅
- B. -InitScript
- C. -StartupScript
- D. -ScriptData

2. 3. **What is the purpose of cloud-init?**

- A. Backup solution
- B. VM metric monitor
- C. Initial configuration on Linux ✅
- D. Azure billing script

4. **Which service is used to view VM performance metrics in Azure?**

- A. Azure Bastion
- B. Azure Monitor ✅
- C. Azure DevOps
- D. Azure Advisor

5. **What is the primary role of a Network Security Group?**

- A. Store VM data
- B. Apply monitoring
- C. Filter network traffic ✅
- D. Create subnets

6. **How do you enable Syslog category in diagnostics via PS?**

- A. -Category Syslog -Enabled $true ✅
- B. -LogCategory Syslog
- C. -EnableSyslog
- D. -Logs Syslog

7. **To view VM public IP via PowerShell, which property is used?**

- A. IpAddress ✅
- B. IPAddressId
- C. PublicIp
- D. Address

8. **Which cmdlet adds a security rule to an existing NSG object?**

- A. Add-AzNetworkSecurityRuleConfig ✅
- B. New-AzNSGRule
- C. Set-AzNetworkRule
- D. Update-AzNSG

### Spaced‑Repetition Plan
**Day 1 Review:**
- Revisit steps 2–4. Draw the architecture from memory.
- Prompt: “List each Azure resource created and its purpose.”

**Week 1 Review:**
- Recreate the lab without notes.
- Prompt: “What steps are needed to ensure a VM is monitored and secured?”

**Month 1 Review:**
- Automate full lab via Azure Automation Runbook or GitHub Actions.
- Prompt: “How would you convert these steps into an ARM template or Bicep?”

