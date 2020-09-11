Issue link: https://github.com/MicrosoftDocs/azure-docs/issues/62449

When I deploy [this ARM template](https://github.com/simonzhaoms/issues/blob/master/azure/arm/deploymentscript/argumentspace/template.json) in `southeastasia` with a subscription named `simon he zhao` (NOTE the name contains spaces):

```
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "variables": {
    "umiName": "[concat(resourceGroup().name, 'umi')]",
    "umiId": "[resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', variables('umiName'))]",
    "roleAssignName": "[guid(resourceGroup().id, variables('umiName'))]",
    "roleDefIdPre": "[concat(subscription().id, '/providers/Microsoft.Authorization/roleDefinitions/')]",
    "Owner": "[concat(variables('roleDefIdPre'), '8e3af657-a8ff-443c-a75c-2fe8c4bcb635')]"
  },
  "resources": [
    {
      "name": "[variables('umiName')]",
      "type": "Microsoft.ManagedIdentity/userAssignedIdentities",
      "apiVersion": "2018-11-30",
      "location": "[resourceGroup().location]"
    },
    {
      "name": "[variables('roleAssignName')]",
      "type": "Microsoft.Authorization/roleAssignments",
      "apiVersion": "2018-09-01-preview",
      "dependsOn": ["[variables('umiName')]"],
      "properties": {
        "roleDefinitionId": "[variables('Owner')]",
        "principalId": "[reference(variables('umiName')).principalId]",
        "principalType": "ServicePrincipal"
      }
    },
    {
      "name": "[concat(resourceGroup().name, 'deployscript')]",
      "type": "Microsoft.Resources/deploymentScripts",
      "apiVersion": "2019-10-01-preview",
      "dependsOn": [
        "[variables('roleAssignName')]"
      ],
      "identity": {
        "type": "UserAssigned",
        "userAssignedIdentities": {
          "[variables('umiId')]": {}
        }
      },
      "location": "[resourceGroup().location]",
      "kind": "AzureCLI",
      "properties": {
        "azCliVersion": "2.0.80",
        "retentionInterval": "P1D",
        "cleanupPreference": "OnExpiration",
        "primaryScriptUri": "https://raw.githubusercontent.com/simonzhaoms/issues/master/azure/arm/deploymentscript/argumentspace/main.sh",
        "supportingScriptUris": [],
        "arguments": "[concat(resourceGroup().name, ' ', subscription().displayName, ' ', resourceGroup().location)]"
      }
    }
  ]
}
```

where [main.sh](https://github.com/simonzhaoms/issues/blob/master/azure/arm/deploymentscript/argumentspace/main.sh) is:

```
#!/bin/bash -
echo "resource group: $1;"
echo "subscription: $2;"
echo "location: $3;"
```

I got the following logs:

```
resource group: simonzunquote; subscription: simon; location: he;
```

However, the expected logs would be:

```
resource group: simonzunquote; subscription: simon he zhao; location: southeastasia;
```

I tried to quote the subscription name in [this ARM template](https://github.com/simonzhaoms/issues/blob/master/azure/arm/deploymentscript/argumentspace/template_quote.json) with [the only difference from the above template](https://github.com/simonzhaoms/issues/blob/ac1202ba5e82a8298c13c86e67b8e0c0a5ec8d38/azure/arm/deploymentscript/argumentspace/template_quote.json#L50):

```
        "arguments": "[concat(resourceGroup().name, ' \"', subscription().displayName, '\" ', resourceGroup().location)]"
```

But the result was the same.  I took a look at the script `DeploymentScript.sh` stored in the `azscriptinput` folder of the created storage `rndoqwqusiszmazscripts`, found the arguments passed from the ARM template to `main.sh` were assigned to `system_deployment_script_arguments=$@` before `bash $file_to_execute $system_deployment_script_arguments`.  That is, `DeploymentScript.sh` looks like:

```
...
system_deployment_script_arguments=$@
...
bash $file_to_execute $system_deployment_script_arguments
...
```

I think the indirect reference of `$@` causes the problem.  One solution may be using quoted `"$@"` directly:

```
...
bash $file_to_execute "$@"
...
```

or storing `$@` as an array:

```
system_deployment_script_arguments=("$@")
...
bash $file_to_execute "${system_deployment_script_arguments[@]}"
...
```


---
#### Document Details

⚠ *Do not edit this section. It is required for docs.microsoft.com ➟ GitHub issue linking.*

* ID: b5c4c8c5-7757-a969-5800-17f2f2312649
* Version Independent ID: 9ced8dd5-43ca-6d1f-c054-842299db8327
* Content: [Use deployment scripts in templates - Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-script-template?tabs=CLI)
* Content Source: [articles/azure-resource-manager/templates/deployment-script-template.md](https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/azure-resource-manager/templates/deployment-script-template.md)
* Service: **azure-resource-manager**
* Sub-service: **templates**
* GitHub Login: @mumian
* Microsoft Alias: **jgao**
