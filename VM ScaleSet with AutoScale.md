# ARM template to deploy VM Scale set with autoscale

```json
{
  "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmSku": {
      "type": "string",
      "defaultValue": "Standard_D1",
      "metadata": {
        "description": "Size of VMs in the VM Scale Set."
      }
    },
    "sourceImageVhdUri": {
      "type": "string",
      "metadata": {
        "description": "The source of the blob containing the custom image"
      }
    },
    "vmssName": {
      "type": "string",
      "metadata": {
        "description": "String used as a base for naming resources. Must be 3-61 characters in length and globally unique across Azure. A hash is prepended to this string for some resources, and resource-specific information is appended."
      },
      "maxLength": 61
    },
    "instanceCount": {
      "type": "int",
      "metadata": {
        "description": "Number of VM instances (100 or less)."
      },
      "defaultValue": 2,
      "maxValue": 100
    },
    "adminUsername": {
      "type": "string",
      "metadata": {
        "description": "Admin username on all VMs."
      }
    },
    "adminPassword": {
      "type": "securestring",
      "metadata": {
        "description": "Admin password on all VMs."
      }
    },
    "existingSubnetResourceId": {
      "type": "string",
      "metadata": {
        "description": "Resource ID of the existing subnet to deploy the scale set into. Should be of the form: /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP/providers/Microsoft.Network/virtualNetworks/YOUR_VNET_NAME/subnets/YOUR_SUBNET_NAME"
      }
    }
  },
  "variables": {
    "namingInfix": "[toLower(substring(concat(parameters('vmssName'), uniqueString(resourceGroup().id)), 0, 9))]",
    "storageAccountType": "Standard_LRS",
	"newStorageAccountSuffix": "[concat('shakti','sa')]",

    "uniqueStringArray": [
      "[concat(uniqueString(concat(resourceGroup().id, variables('newStorageAccountSuffix'), '0')))]",
      "[concat(uniqueString(concat(resourceGroup().id, variables('newStorageAccountSuffix'), '1')))]",
      "[concat(uniqueString(concat(resourceGroup().id, variables('newStorageAccountSuffix'), '2')))]",
      "[concat(uniqueString(concat(resourceGroup().id, variables('newStorageAccountSuffix'), '3')))]",
      "[concat(uniqueString(concat(resourceGroup().id, variables('newStorageAccountSuffix'), '4')))]"
    ],
    "saCount": "[length(variables('uniqueStringArray'))]",
    "publicIPAddressName": "pip",
    "loadBalancerName": "loadBalancer",
    "loadBalancerFrontEndName": "loadBalancerFrontEnd",
    "loadBalancerBackEndName": "loadBalancerBackEnd",
    "loadBalancerProbeName": "loadBalancerHttpProbe",
    "loadBalancerNatPoolName": "loadBalancerNatPool",
    "computeApi": "2016-03-30"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "name": "[concat(variables('uniqueStringArray')[copyIndex()], variables('newStorageAccountSuffix'))]",
      "location": "[resourceGroup().location]",
      "apiVersion": "2015-06-15",
      "copy": {
        "name": "storageLoop",
        "count": "[variables('saCount')]"
      },
      "properties": {
        "accountType": "[variables('storageAccountType')]"
      }
    },
    {
      "type": "Microsoft.Compute/virtualMachineScaleSets",
      "apiVersion": "[variables('computeApi')]",
      "name": "[parameters('vmssName')]",
      "location": "[resourceGroup().location]",
      "dependsOn": [
        "storageLoop",
        "[concat('Microsoft.Network/loadBalancers/',variables('loadBalancerName'))]"
      ],
      "sku": {
        "name": "[parameters('vmSku')]",
        "tier": "Standard",
        "capacity": "[parameters('instanceCount')]"
      },
      "properties": {
        "overprovision": "true",
        "upgradePolicy": {
          "mode": "Manual"
        },
        "virtualMachineProfile": {
          "storageProfile": {
            "osDisk": {
              "name": "vmssosdisk",
              "caching": "ReadOnly",
              "createOption": "FromImage",
              "osType": "Windows",
              "image": {
                "uri": "[parameters('sourceImageVhdUri')]"
              }
            }
          },
          "osProfile": {
            "computerNamePrefix": "[parameters('vmSSName')]",
            "adminUsername": "[parameters('adminUsername')]",
            "adminPassword": "[parameters('adminPassword')]"
          },
          "networkProfile": {
            "networkInterfaceConfigurations": [
              {
                "name": "nic",
                "properties": {
                  "primary": "true",
                  "ipConfigurations": [
                    {
                      "name": "ipconfig",
                      "properties": {
                        "subnet": {
                          "id": "[parameters('existingSubnetResourceId')]"
                        },
                        "loadBalancerBackendAddressPools": [
                          {
                            "id": "[concat('/subscriptions/', subscription().subscriptionId,'/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Network/loadBalancers/', variables('loadBalancerName'), '/backendAddressPools/', variables('loadBalancerBackEndName'))]"
                          }
                        ],
                        "loadBalancerInboundNatPools": [
                          {
                            "id": "[concat('/subscriptions/', subscription().subscriptionId,'/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Network/loadBalancers/', variables('loadBalancerName'), '/inboundNatPools/', variables('loadBalancerNatPoolName'))]"
                          }
                        ]
                      }
                    }
                  ]
                }
              }
            ]
          }
        }
      }
    },
    {
      "type": "Microsoft.Network/publicIPAddresses",
      "name": "[variables('publicIPAddressName')]",
      "location": "[resourceGroup().location]",
      "apiVersion": "2016-06-01",
      "properties": {
        "publicIPAllocationMethod": "Dynamic",
        "dnsSettings": {
          "domainNameLabel": "[toLower(parameters('vmssName'))]"
        }
      }
    },
    {
      "type": "Microsoft.Network/loadBalancers",
      "name": "[variables('loadBalancerName')]",
      "location": "[resourceGroup().location]",
      "apiVersion": "2016-06-01",
      "dependsOn": [
        "[concat('Microsoft.Network/publicIPAddresses/', variables('publicIPAddressName'))]"
      ],
      "properties": {
        "frontendIPConfigurations": [
          {
            "name": "[variables('loadBalancerFrontEndName')]",
            "properties": {
              "publicIPAddress": {
                "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('publicIPAddressName'))]"
              }
            }
          }
        ],
        "backendAddressPools": [
          {
            "name": "[variables('loadBalancerBackendName')]"
          }
        ],
        "loadBalancingRules": [
          {
            "name": "roundRobinLBRule",
            "properties": {
              "frontendIPConfiguration": {
                "id": "[concat(resourceId('Microsoft.Network/loadBalancers', variables('loadBalancerName')), '/frontendIPConfigurations/', variables('loadBalancerFrontEndName'))]"
              },
              "backendAddressPool": {
                "id": "[concat(resourceId('Microsoft.Network/loadBalancers', variables('loadBalancerName')), '/backendAddressPools/', variables('loadBalancerBackendName'))]"
              },
              "protocol": "tcp",
              "frontendPort": 80,
              "backendPort": 80,
              "enableFloatingIP": false,
              "idleTimeoutInMinutes": 5,
              "probe": {
                "id": "[concat(resourceId('Microsoft.Network/loadBalancers', variables('loadBalancerName')), '/probes/', variables('loadBalancerProbeName'))]"
              }
            }
          }
        ],
        "probes": [
          {
            "name": "[variables('loadBalancerProbeName')]",
            "properties": {
              "protocol": "tcp",
              "port": 80,
              "intervalInSeconds": "5",
              "numberOfProbes": "2"
            }
          }
        ],
        "inboundNatPools": [
          {
            "name": "[variables('loadBalancerNatPoolName')]",
            "properties": {
              "frontendIPConfiguration": {
                "id": "[concat(resourceId('Microsoft.Network/loadBalancers', variables('loadBalancerName')), '/frontendIPConfigurations/', variables('loadBalancerFrontEndName'))]"
              },
              "protocol": "tcp",
              "frontendPortRangeStart": "50000",
              "frontendPortRangeEnd": "50019",
              "backendPort": "3389"
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Insights/autoscaleSettings",
      "apiVersion": "2015-04-01",
      "name": "Autoscale",
      "location": "[resourceGroup().location]",
      "dependsOn": [
        "[concat('Microsoft.Compute/virtualMachineScaleSets/',parameters('vmssName'))]"
      ],
      "properties": {
        "enabled": true,
        "name": "Autoscale",
        "profiles": [
          {
            "name": "Profile1",
            "capacity": {
              "minimum": "1",
              "maximum": "10",
              "default": "1"
            },
            "rules": [
              {
                "metricTrigger": {
                  "metricName": "Percentage CPU",
                  "metricNamespace": "",
                  "metricResourceUri": "[concat('/subscriptions/', subscription().subscriptionId, '/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Compute/virtualMachineScaleSets/', parameters('vmssName'))]",
                  "timeGrain": "PT1M",
                  "statistic": "Average",
                  "timeWindow": "PT5M",
                  "timeAggregation": "Average",
                  "operator": "GreaterThan",
                  "threshold": 75
                },
                "scaleAction": {
                  "direction": "Increase",
                  "type": "ChangeCount",
                  "value": "1",
                  "cooldown": "PT5M"
                }
              },
              {
                "metricTrigger": {
                  "metricName": "Percentage CPU",
                  "metricNamespace": "",
                  "metricResourceUri": "[concat('/subscriptions/', subscription().subscriptionId, '/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Compute/virtualMachineScaleSets/', parameters('vmssName'))]",
                  "timeGrain": "PT1M",
                  "statistic": "Average",
                  "timeWindow": "PT5M",
                  "timeAggregation": "Average",
                  "operator": "LessThan",
                  "threshold": 25
                },
                "scaleAction": {
                  "direction": "Decrease",
                  "type": "ChangeCount",
                  "value": "1",
                  "cooldown": "PT5M"
                }
              }
            ]
          }
        ],
        "targetResourceUri": "[concat('/subscriptions/', subscription().subscriptionId, '/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Compute/virtualMachineScaleSets/', parameters('vmssName'))]"
      }
    }
  ]
}
```