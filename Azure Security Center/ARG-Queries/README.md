# Starter Kit - ARG Queries for Azure Security Center Recommendations
Azure Resource Graph (ARG) provides an efficient way to query at scale across a given set of subscriptions for any Azure Resource (for more information please visit https://docs.microsoft.com/en-us/azure/governance/resource-graph/). 
A particular usfeul use case is using ARG to query, visualize or export Azure Security Center (ASC) recommendations in order to get the information that matter most to you.

This starter kit consists of a set of basic ARG queries that have been created to help anyone build on top of them based on different needs and requirements.

1. Get ASC recommendations in usefull format
```
securityresource
 | where type == "microsoft.security/assessments"
 // Get recommendations in useful format
 | project
    ['TenantID'] = tenantId,
    ['SubscriptionID'] = subscriptionId,
    ['AssessmentID'] = name,
    ['DisplayName'] = properties.displayName,
    ['ResourceType'] = tolower(split(properties.resourceDetails.Id,"/").[7]),
    ['ResourceName'] = tolower(split(properties.resourceDetails.Id,"/").[8]),
    ['ResourceGroup'] = resourceGroup,
    ['ContainsNestedRecom'] = tostring(properties.additionalData.subAssessmentsLink),
    ['StatusCode'] = properties.status.code,
    ['StatusDescription'] = properties.status.description,
    ['PolicyDefID'] = properties.metadata.policyDefinitionId,
    ['Description'] = properties.metadata.description,
    ['RecomType'] = properties.metadata.assessmentType,
    ['Remediation'] = properties.metadata.remediationDescription,
    ['RemediationEffort'] = properties.metadata.implementationEffort,
    ['Severity'] = properties.metadata.severity,
    ['Categories'] = properties.metadata.categories,
    ['UserImpact'] = properties.metadata.userImpact,
    ['Threats'] = properties.metadata.threats,
    ['Link'] = properties.links.azurePortal
```
2. Get ASC nested recommendations in useful format
```
securityresources
 | where type == "microsoft.security/assessments/subassessments"
 // Get recommendations in useful format
 | project 
    ['TenantID'] = tenantId,
    ['SubscriptionID'] = subscriptionId,
    ['ParentAssessmentID'] = split(id,"/").[12],
    ['NestedAssessmentID'] = split(id,"/").[14],
    ['NestedDescription'] = properties.description,
    ['NestedDisplayName'] = properties.displayName,
    ['ResourceType'] = tolower(split(id,"/").[7]),
    ['ResourceName'] = tolower(split(id,"/").[8]),
    ['ResourceGroup'] = resourceGroup,
    ['NestedCategory'] = properties.category,
    ['TimeGenerated'] = properties.timeGenerated,
    ['NestedRemediation'] = properties.remediation,
    ['NestedImpact'] = properties.impact,
    ['NestedAdditionalData'] = properties.additionalData
```
3. Get ASC parent + nested recommendations combined (best to filter by resource or resource group to get less result)
```
securityresources
 | where type == "microsoft.security/assessments"
 // Get recommendations in useful format
 | project
    ['ParentAssessmentID'] = name,
    ['TenantID'] = tenantId,
    ['SubscriptionID'] = subscriptionId,
    ['DisplayName'] = properties.displayName,
    ['ResourceType'] = tolower(split(properties.resourceDetails.Id,"/").[7]),
    ['ResourceName'] = tolower(split(properties.resourceDetails.Id,"/").[8]),
    ['ResourceGroup'] = resourceGroup,
    ['StatusCode'] = properties.status.code,
    ['StatusDescription'] = properties.status.description,
    ['PolicyDefID'] = properties.metadata.policyDefinitionId,
    ['Description'] = properties.metadata.description,
    ['RecomType'] = properties.metadata.assessmentType,
    ['Remediation'] = properties.metadata.remediationDescription,
    ['RemediationEffort'] = properties.metadata.implementationEffort,
    ['Severity'] = properties.metadata.severity,
    ['Categories'] = properties.metadata.categories,
    ['UserImpact'] = properties.metadata.userImpact,
    ['Threats'] = properties.metadata.threats,
    ['Link'] = properties.links.azurePortal
    // Filter to test with less results
        // | where ResourceName == "ntierapp-vm-web"
    // Joining Parentassessment + Nestedassessment table
    | join kind=leftouter (
        securityresources
            | where type == "microsoft.security/assessments/subassessments"
            | extend ['ParentAssessmentID'] = tostring(split(id,"/").[12]),
                     ['ResourceName'] = tolower(split(id,"/").[8]),
                     ['NestedAssessmentID'] = tostring(split(id,"/").[14]),
                     ['NestedDescription'] = properties.description,
                     ['NestedDisplayName'] = properties.displayName,
                     ['NestedCategory'] = properties.category,
                     ['TimeGenerated'] = properties.timeGenerated,
                     ['NestedRemediation'] = properties.remediation,
                     ['NestedImpact'] = properties.impact,
                     ['NestedAdditionalData'] = properties.additionalData
                     //| where ResourceName == "ntierapp-vm-web"
        ) on ParentAssessmentID
        | project TenantID, SubscriptionID, ParentAssessmentID, NestedAssessmentID, DisplayName, NestedDisplayName, Description, NestedDescription, ResourceType, ResourceName, ResourceGroup, StatusCode, StatusDescription, RecomType, Severity, Threats, RemediationEffort, Remediation, NestedRemediation, UserImpact, NestedImpact, Categories, NestedCategory, TimeGenerated, PolicyDefID, Link
```
4. Get ASC Current Pricing Tiers
```
securityresources
| where type == "microsoft.security/pricings"
// Project in useful format
| project 
    ['SubscriptionID'] = subscriptionId,
    ['TenantID'] = tenantId,
    ['Service'] = name,
    ['PricingTier'] = properties.pricingTier,
    ['FreeTrialRemainingTime'] = properties.freeTrialRemainingTime
// filter by subscription
// | where SubscriptionID == '375bb878-6089-40b2-8a4d-7bb63807d186'
```
5. Create your own Dashboard or visualization charts

One instersting feature of ARG is that apart from querying and exporting the information, you can created charts and dashboards directly from Azure Resource Graph Explorer and pin them on your Azure Portal Dashboards. For this particular examples, we want to retrieve all the recomendations with High severity and Low remediation effort provided by ASC, focussing on Threat vectors and impacted resources. For all the dashaboards created (see below screenshot) we have used the following ARG query:
```
securityresources
 | where type == "microsoft.security/assessments"
 // Get recommendations in useful format
 | project
    ['TenantID'] = tenantId,
    ['SubscriptionID'] = subscriptionId,
    ['AssessmentID'] = name,
    ['DisplayName'] = properties.displayName,
    ['ResourceType'] = tolower(split(properties.resourceDetails.Id,"/").[7]),
    ['ResourceName'] = tolower(split(properties.resourceDetails.Id,"/").[8]),
    ['ResourceGroup'] = resourceGroup,
    ['ContainsNestedRecom'] = tostring(properties.additionalData.subAssessmentsLink),
    ['StatusCode'] = properties.status.code,
    ['StatusDescription'] = properties.status.description,
    ['PolicyDefID'] = properties.metadata.policyDefinitionId,
    ['Description'] = properties.metadata.description,
    ['RecomType'] = properties.metadata.assessmentType,
    ['Remediation'] = properties.metadata.remediationDescription,
    ['RemediationEffort'] = properties.metadata.implementationEffort,
    ['Severity'] = properties.metadata.severity,
    ['Categories'] = properties.metadata.categories,
    ['UserImpact'] = properties.metadata.userImpact,
    ['Threats'] = properties.metadata.threats,
    ['Link'] = properties.links.azurePortal
// Filter
 | where Severity == "High"
 | where RemediationEffort == "Low"
 // | where ResourceName == "ntierapp-vm-web"
// summarize and order
 | summarize count() by tostring(Threats)
 | order by count_
 ```
 By tweking the previous query to provide different dashboards, we have benn able to set the following dashboard that can now be consumed in a very efficient manner each time I log onto my Azure Portal.
 
![Image of ASCRecom-CustomDashboard](https://github.com/carlosfar/public/blob/master/Azure%20Security%20Center/ARG-Queries/ASCRecommendationsCustomDashboard.png)
