# Santander Security Guardrails - Azure Policy Style guide

Azure Policy is a service in Azure that can be used to create, assign, and manage policies and security guardrails. These policies enforce different rules and effects over the Azure resources, so those resources stay compliant with Santander Security Guardrails, standards and service level agreements. Azure Policy meets this need by evaluating Azure resources for non-compliance with assigned policies. All data stored by Azure Policy is encrypted at rest.

Understanding how to create and manage policies in Azure is important for staying compliant with security standards proposed by Santander Security Guardrails. A custom policy definition allows customers to define their own rules for using Azure. 
Before creating a custom policy, check [Microsoft built-in policies](https://github.com/Azure/azure-policy/tree/master/built-in-policies) and [Microsoft policy samples](https://github.com/Azure/azure-policy/tree/master/samples) to see if a policy that matches your needs already exists.
The steps for defining a custom Azure Policy are mostly the same for each Azure component.

# Create an Azure Policy definition

The approach to creating a custom policy follows these steps:

- [Identify business requirements](#Prerequisites)
- [Map each requirement to an Azure resource property](#Determine-resource-properties)
- [Map the property to an alias](#Find-the-property-alias)
- [Determine which effect to use](#Determine-the-effect-to-use)
- [Compose the policy definition](#Compose-the-definition)

## Prerequisites

Before creating the policy definition, it's important to understand the intent of the policy. The requirements should clearly identify both the "to be" and the "not to be" resource states.
You must define the expected state of the resource, but you need to define also what to do with the non-compliant resources. 
Azure Policy supports a number of effects, but it’s recommended to use Audit as initial effect in all Azure Policy definitions to have compliance reports and avoid interrupt production. Other option for an **Audit** effect could be **AuditIfNotExists**. The full set of Azure Policy effects can be found in https://docs.microsoft.com/en-us/azure/governance/policy/concepts/effects

## Determine resource properties
Based on the business requirement for the Azure Policy, identify the required resources to audit with Azure Policy to know the properties to use in the policy definition. Azure Policy evaluates against the JSON representation of the resource, so we'll need to understand the properties available on the resources.
There are many ways to determine the properties for an Azure resource:

* Azure Policy extension for VS Code
* Resource Manager templates
    * Export existing resource
    * Creation experience
    * Quickstart templates (GitHub)
    * Template reference docs
* Azure Resource Explorer

### View resources in VS Code extension

The VS Code extension can be used to browse resources in the required environment and see the Resource Manager properties on each resource.

### Resource Manager templates

There are several ways to look at a Resource Manager template that includes the property you're looking to manage.

#### Existing resource in the portal
The simplest way to find properties is to look at an existing resource of the same type. Resources already configured with the setting you want to enforce also provide the value to compare against. Look at the Export template page (under Settings) in the Azure portal for that specific resource.
This is an example for a storage account:

```
...
"resources": [{
    "comments": "Generalized from resource: '/subscriptions/{subscriptionId}/resourceGroups/myResourceGroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount'.",
    "type": "Microsoft.Storage/storageAccounts",
    "sku": {
        "name": "Standard_LRS",
        "tier": "Standard"
    },
    "kind": "Storage",
    "name": "[parameters('storageAccounts_mystorageaccount_name')]",
    "apiVersion": "2018-07-01",
    "location": "westus",
    "tags": {
        "ms-resource-usage": "azure-cloud-shell"
    },
    "scale": null,
    "properties": {
        "networkAcls": {
            "bypass": "AzureServices",
            "virtualNetworkRules": [],
            "ipRules": [],
            "defaultAction": "Allow"
        },
        "supportsHttpsTrafficOnly": false,
        "encryption": {
            "services": {
                "file": {
                    "enabled": true
                },
                "blob": {
                    "enabled": true
                }
            },
            "keySource": "Microsoft.Storage"
        }
    },
    "dependsOn": []
}]
...    
```

#### Create a resource in the portal

Another way through the portal is the resource creation experience. 
On the Review + create tab after craating a resource, a link is at the bottom of the page to Download a template for automation. Selecting the link opens the template that creates the resource we configured. 

#### Quickstart templates on GitHub

The [Azure quickstart templates](https://github.com/Azure/azure-quickstart-templates) on GitHub has hundreds of Resource Manager templates built for different resources. These templates can be a great way to find the resource property you're looking for. Some properties may appear to be what you're looking for, but control something else.

#### Resource reference docs

To validate the correct property for each resource, check the [Resource Manager template reference](https://docs.microsoft.com/en-us/azure/templates/) and search for the resource you are looking for. The properties object has a list of valid parameters. Check the list to look for a property to meet the business requirements.

### Azure Resource Explorer

Another way to explore your Azure resources is through the Azure Resource Explorer (Preview). This tool uses the context of your subscription, so you need to authenticate to the [Azure Resorce Explorer website](https://resources.azure.com/) with your Azure credentials. Once authenticated, you can browse by providers, subscriptions, resource groups, and resources.
Locate an appropiate resource and look at the properties. Selecting the Documentation tab, you can check that the property description matches your requirements.

## Find the property alias

Once identified the resource property, it is needed to map that property to an alias. 
You use property aliases to access specific properties for a resource type. Aliases enable you to restrict what values or conditions are allowed for a property on a resource. Each alias maps to paths in different API versions for a given resource type.
There are a few ways to determine the aliases for an Azure resource:
* Azure Policy extension for VS Code
* Azure CLI
* Azure PowerShell
* Azure Resource Graph

### Get aliases in VS Code extension

The Azure Policy extension for VS Code extension makes it easy to browse your resources and discover aliases. 
When a resource is selected, whether through the search interface or by selecting it in the treeview, the Azure Policy extension opens the JSON file representing that resource and all it's Resource Manager property values.
Once a resource is open, hovering over the Resource Manager property name or value displays the Azure Policy alias if one exists.

### Azure CLI

In Azure CLI, the az provider command group is used to search for resource aliases. The following example is a filter for the Microsoft.Storage namespace aliases.
#Login first with az login if not using Cloud Shell

#Get Azure Policy aliases for type Microsoft.Storage

az provider show --namespace Microsoft.Storage --expand "resourceTypes/aliases" --query "resourceTypes[].aliases[].name"

### Azure PowerShell
In Azure PowerShell, the Get-AzPolicyAlias cmdlet is used to search for resource aliases. The following example is a filter for the Microsoft.Storage namespace aliases.

#Login first with Connect-AzAccount if not using Cloud Shell

#Use Get-AzPolicyAlias to list aliases for Microsoft.Storage

(Get-AzPolicyAlias -NamespaceMatch 'Microsoft.Storage').Aliases

### Azure Resource Graph

Azure Resource Graph (https://docs.microsoft.com/en-us/azure/governance/resource-graph/overview) is a service that provides another method to find properties of Azure resources. Here is a sample query for looking at a single storage account with Resource Graph:

Kusto
```
Resources
| where type=~'microsoft.storage/storageaccounts'
| limit 1
```
Azure CLI
```
az graph query -q "Resources | where type=~'microsoft.storage/storageaccounts' | limit 1"
Azure PowerShellCopy

Search-AzGraph -Query "Resources | where type=~'microsoft.storage/storageaccounts' | limit 1"
Azure Resource Graph results can also include alias details by projecting the aliases array:
```

Kusto
```
Resources
| where type=~'microsoft.storage/storageaccounts'
| limit 1
| project aliases
```

Azure CLI
```
az graph query -q "Resources | where type=~'microsoft.storage/storageaccounts' | limit 1 | project aliases"
Azure PowerShellCopy
Try It
Search-AzGraph -Query "Resources | where type=~'microsoft.storage/storageaccounts' | limit 1 | project aliases"
```

## Determine the effect to use

Deciding what to do with your non-compliant resources is very important. Each possible response to a non-compliant resource is called an effect. The effect controls if the non-compliant resource is logged, blocked, has data appended, or has a deployment associated to it for putting the resource back into a compliant state.
Audit is a good first choice for a policy effect to determine what the impact of a policy is before setting it to Enforce any resource configuration. 
It is highly recommended to use Audit or AuditIfNotExists as first choice in every Azure Policy definition to check the desired configuration.

## Compose the definition

Once you have the property details and alias for what you plan to manage, the next step is to compose the policy rule itself. If you aren't yet familiar with the policy language, check policy definition structure (https://docs.microsoft.com/en-us/azure/governance/policy/concepts/definition-structure) for how to structure the policy definition. 
Here is an empty template of what a policy definition looks like:

```
{
    "properties": {
        "displayName": "<displayName>",
        "description": "<description>",
        "mode": "<mode>",
        "parameters": {
                <parameters>
        },
        "policyRule": {
            "if": {
                <rule>
            },
            "then": {
                "effect": "<effect>"
            }
        }
    }
}
```

### Metadata

The first three components are policy metadata. Mode is primarily about tags and resource location. Use the value Indexed if you want to limit evaluation to resources that support tags, or use the all value if you don’t want to limit evaluation and check all resources.

JSON

```
"displayName": "Deny storage accounts not using only HTTPS",
"description": "Deny storage accounts not using only HTTPS. Checks the supportsHttpsTrafficOnly property on StorageAccounts.",
"mode": "all",
```

### Parameters

Parameters help simplify the policy management by reducing the number of policy definitions. Think of parameters like the fields on a form – name, address, city, state. These parameters always stay the same, however their values change based on the individual filling out the form. Parameters work the same way when building policies. By including parameters in a policy definition, you can reuse that policy for different scenarios by using different values.
Use parameters whenever possible to avoid modify Azure Policy definition code when assigning Azure Policy definition to different resources, increasing the flexibility of the policy definition during policy assignment. 
Example: A parameter value can be passed to a tag field

### Policy rule

Composing the policy rule is the final step in building a custom policy definition. 
The policy rule consists of If and Then blocks. In the If block, you define one or more conditions that specify when the policy is enforced. You can apply logical operators to these conditions to precisely define the scenario for a policy.
In the Then block, you define the effect that happens when the If conditions are fulfilled.

JSON

```
{
    "if": {
        <condition> | <logical operator>
    },
    "then": {
        "effect": "deny | audit | append | auditIfNotExists | deployIfNotExists | disabled"
    }
}
```

#### Logical operators

Supported logical operators are:
* "not": {condition or operator}
* "allOf": [{condition or operator},{condition or operator}]
* "anyOf": [{condition or operator},{condition or operator}]
The not syntax inverts the result of the condition. The allOf syntax (similar to the logical And operation) requires all conditions to be true. The anyOf syntax (similar to the logical Or operation) requires one or more conditions to be true.
You can nest logical operators.

#### Conditions

A condition evaluates whether a field or the value accessor meets certain criteria. The supported conditions are:
- "equals": "stringValue"
- "notEquals": "stringValue"
- "like": "stringValue"
- "notLike": "stringValue"
- "match": "stringValue"
- "matchInsensitively": "stringValue"
- "notMatch": "stringValue"
- "notMatchInsensitively": "stringValue"
- "contains": "stringValue"
- "notContains": "stringValue"
- "in": ["stringValue1","stringValue2"]
- "notIn": ["stringValue1","stringValue2"]
- "containsKey": "keyName"
- "notContainsKey": "keyName"
- "less": "value"
- "lessOrEquals": "value"
- "greater": "value"
- "greaterOrEquals": "value"
- "exists": "bool"

When using the like and notLike conditions, you provide a wildcard * in the value. The value shouldn't have more than one wildcard *.
When using the match and notMatch conditions, provide # to match a digit, ? for a letter, . to match any character, and any other character to match that actual character. While, match and notMatch are case-sensitive, all other conditions that evaluate a stringValue are case-insensitive. Case-insensitive alternatives are available in matchInsensitively and notMatchInsensitively.
In an [*] alias array field value, each element in the array is evaluated individually with logical and between elements. For more information, see Evaluating the [*] alias.

#### Fields

Conditions are formed by using fields. A field matches properties in the resource request payload and describes the state of the resource.
The following fields are supported:
- name
- fullName
    - Returns the full name of the resource. The full name of a resource is the resource name prepended by any parent resource names (for example "myServer/myDatabase").
- kind
- type
- location
    - Use global for resources that are location agnostic.
- identity.type
    - Returns the type of managed identity enabled on the resource.
- tags
- tags['<tagName>']
    - This bracket syntax supports tag names that have punctuation such as a hyphen, period, or space.
    - Where <tagName> is the name of the tag to validate the condition for.
    - Examples: tags['Acct.CostCenter'] where Acct.CostCenter is the name of the tag.
- tags['''<tagName>''']
    - This bracket syntax supports tag names that have apostrophes in it by escaping with double apostrophes.
    - Where '<tagName>' is the name of the tag to validate the condition for.
    - Example: tags['''My.Apostrophe.Tag'''] where 'My.Apostrophe.Tag' is the name of the tag.
- property aliases - for a list, see Aliases.

#### Value

Conditions can also be formed using value. Value checks conditions against parameters, supported template functions, or literals. value is paired with any supported condition.

#### Count

Conditions that count how many members of an array in the resource payload satisfy a condition expression can be formed using count expression. Common scenarios are checking whether 'at least one of', 'exactly one of', 'all of', or 'none of' the array members satisfy the condition. count evaluates each [*] alias array member for a condition expression and sums the true results, which is then compared to the expression operator.

#### Effect

Azure Policy supports the following types of effect:
- Append: adds the defined set of fields to the request
- Audit: generates a warning event in activity log but doesn't fail the request
- AuditIfNotExists: generates a warning event in activity log if a related resource doesn't exist
- Deny: generates an event in the activity log and fails the request
- DeployIfNotExists: deploys a related resource if it doesn't already exist
- Disabled: doesn't evaluate resources for compliance to the policy rule
- EnforceOPAConstraint (preview): configures the Open Policy Agent admissions controller with Gatekeeper v3 for self-managed Kubernetes clusters on Azure (preview)
- EnforceRegoPolicy (preview): configures the Open Policy Agent admissions controller with Gatekeeper v2 in Azure Kubernetes Service
- Modify: adds, updates, or removes the defined tags from a resource

#### Policy functions

All Resource Manager template functions are available to use within a policy rule, except the following functions and user-defined functions:
- copyIndex()
- deployment()
- list*
- newGuid()
- pickZones()
- providers()
- reference()
- resourceId()
- variables()
The following function is available to use in a policy rule, but differs from use in an Azure Resource Manager template:
- utcNow() - Unlike a Resource Manager template, this can be used outside defaultValue.
    - Returns a string that is set to the current date and time in Universal ISO 8601 DateTime format 'yyyy-MM-ddTHH:mm:ss.fffffffZ'
The following functions are only available in policy rules:
- addDays(dateTime, numberOfDaysToAdd)
    - dateTime: [Required] string - String in the Universal ISO 8601 DateTime format 'yyyy-MM-ddTHH:mm:ss.fffffffZ'
    - numberOfDaysToAdd: [Required] integer - Number of days to add
- field(fieldName)
    - fieldName: [Required] string - Name of the field to retrieve
    - Returns the value of that field from the resource that is being evaluated by the If condition
    - field is primarily used with AuditIfNotExists and DeployIfNotExists to reference fields on the resource that are being evaluated. An example of this use can be seen in the DeployIfNotExists example.
- requestContext().apiVersion
    - Returns the API version of the request that triggered policy evaluation (example: 2019-09-01). This will be the API version that was used in the PUT/PATCH request for evaluations on resource creation/update. The latest API version is always used during compliance evaluation on existing resources.


### Completed definition

With all three parts of the policy defined (Metadata + Parameters + Policy Rule) you have a completed definition:
The completed definition can be used to create a new policy. Portal and each SDK (Azure CLI, Azure PowerShell, and REST API) accept the definition in different ways, so review the commands for each to validate correct usage. Then assign it, using the parameterized effect, to appropriate resources to manage the security of your resources.

