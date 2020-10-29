---
layout: post
title: How to use ARM template to intergrate vNet into an App Service
---

## Variables

```json
"subnetId": "[resourceId('Microsoft.Network/virtualNetworks/subnets',variables('virtualNetworkName'),parameters('subnetName'))]"
```

## Within Resources section

### vNet definition

```json
{
    "name": "[variables('virtualNetworkName')]",
    "type": "Microsoft.Network/VirtualNetworks",
    "apiVersion": "2019-09-01",
    "location": "[resourceGroup().location]",
    "dependsOn": [],
    "tags": "[parameters('tags')]",
    "properties": {
        "addressSpace": {
            "addressPrefixes": [
                "10.0.0.0/16"
            ]
        },

        "subnets": [
            {
                "name": "[parameters('subnetName')]",
                "properties": {
                    "addressPrefix": "10.0.0.0/24",
                    "serviceEndpoints": [
                        {
                            "service": "Microsoft.Storage",
                            "locations": [
                                "[resourceGroup().location]"
                            ]
                        }
                    ],
                    "delegations": [
                        {
                            "name": "Microsoft.Web.serverFarms",
                            "properties": {
                                "serviceName": "Microsoft.Web/serverFarms"
                            }
                        }
                    ]
                }
            }
        ]
    }
}
```

### Function App Definition

```json
{
    "apiVersion": "2018-11-01",
    "name": "[variables('functionappName')]",
    "type": "Microsoft.Web/sites",
    "kind": "functionapp",
    "location": "[resourceGroup().location]",
    "tags": "[parameters('tags')]",
    "dependsOn": [
        "[resourceId('Microsoft.Storage/storageAccounts',variables('storageAccountName'))]",
        "[resourceId('Microsoft.Web/serverfarms',variables('hostingPlanName'))]"
    ],
    "properties": {
        "name": "[variables('functionappName')]",
        "siteConfig": {
            "appSettings": [
                {
                    "name": "FUNCTIONS_EXTENSION_VERSION",
                    "value": "~3"
                },
                {
                    "name": "FUNCTIONS_WORKER_RUNTIME",
                    "value": "powershell"
                },
                {
                    "name": "AzureWebJobsStorage",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=',variables('storageAccountName'),';AccountKey=',listKeys(resourceId('Microsoft.Storage/storageAccounts',variables('storageAccountName')), '2019-06-01').keys[0].value,';EndpointSuffix=','core.windows.net')]"
                }
            ],
            "use32BitWorkerProcess": "[parameters('use32BitWorkerProcess')]",
            "alwaysOn": "[parameters('alwaysOn')]",
            "powerShellVersion": "[parameters('powerShellVersion')]"
        },
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms',variables('hostingPlanName'))]",
        "clientAffinityEnabled": true
    }
}
```

### vNet intergration

```json

{
    "type": "Microsoft.Web/sites/config",
    "name": "[concat(variables('functionappName'),'/virtualNetwork')]",
    "apiVersion": "2018-02-01",
    "location": "[resourceGroup().location]",
    "dependsOn": [
        "[concat('Microsoft.Web/sites/', variables('functionappName'))]"
    ],
    "properties": {
        "subnetResourceId": "[variables('subnetId')]",
        "swiftSupported": true
    }
}
```
