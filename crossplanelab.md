To install azure cli (az)

https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest&pivots=msi-powershell

<img width="1272" height="235" alt="image" src="https://github.com/user-attachments/assets/94fc265d-82da-44ec-8b5a-43d766c2335e" />

## Install Azure CLI on Windows Using PowerShell

Run the following commands in an **elevated PowerShell window (Run as Administrator)**:

```powershell
$ProgressPreference = 'SilentlyContinue'

Invoke-WebRequest -Uri "https://aka.ms/installazurecliwindowsx64" -OutFile ".\AzureCLI.msi"

Start-Process msiexec.exe -Wait -ArgumentList '/I', 'AzureCLI.msi', '/quiet'

Remove-Item ".\AzureCLI.msi"
```

Verify the installation:

```powershell
az version
```

Login to Azure:

```powershell
 az login --use-device-code
```


To list Azure subscriptions using Azure CLI:

```bash
az account list --output table
```

Show current active subscription

```
az account show --output table
```

If the resource group doesn't exist yet, create it first:
```bash
az group create --name rg-crossplane --location centralindia
```

create a basic single-node AKS cluster in Central India:
```bash
az aks create \
  --resource-group rg-crossplane \
  --name crossplaneaks \
  --node-count 1 \
  --location centralindia \
  --generate-ssh-keys
```

```bash
$ az aks get-credentials --resource-group rg-crossplane --name crossplaneaks
Merged "crossplaneaks" as current context in C:\Users\abc\.kube\config
```

```bash
$ kubectl get node
NAME                                STATUS   ROLES    AGE     VERSION
aks-nodepool1-11095005-vmss000000   Ready    <none>   2m49s   v1.35.6
```

Install helm:

```bash
$helmVersion = "v3.19.0"
Invoke-WebRequest -Uri "https://get.helm.sh/helm-$helmVersion-windows-amd64.zip" -OutFile "$env:TEMP\helm.zip"
Expand-Archive -Path "$env:TEMP\helm.zip" -DestinationPath "$env:TEMP\helm" -Force
Copy-Item "$env:TEMP\helm\windows-amd64\helm.exe" "C:\Windows\System32\helm.exe"
helm version
```



