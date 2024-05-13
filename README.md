
# AzureDevops Releases and Builds Integration Dynatrace

This projects has the purpose to help you integrate your AzureDevops Releases and Builds with Dynatrace in order to visualize statistics, execution logs and alerts.
## Requirements

- Access to your Dynatrace Tenant and permission to create tokens
- Your tenant ID. Which can be found on your environment URL. https://<YOUR_TENANT_ID>.live.dynatrace.com/
- A token with **Ingest Logs v2** scope
- A token with **Write Settings** scope

## Creating Webhooks in AzureDevops

 - First, we need to create two Service Hooks Subscriptions on Azure. One for Builds Completed and one for Release Deployment Completed. 
	 - You can create it on https://{orgName}/{project_name}/_settings/serviceHooks
	 - https://learn.microsoft.com/en-us/azure/devops/service-hooks/services/webhooks?toc=%2Fazure%2Fdevops%2Fmarketplace-extensibility%2Ftoc.json&view=azure-devops
 - During the configuration, do not apply any filter
 - In the settings page of the Subscription, you need to fill the following fields accordantly
	 - URL: https://<YOUR_TENANT_ID>.live.dynatrace.com/api/v2/logs/ingest 
	 - HTTP Headers:
		 - Authorization: Api-token <YOUR_LOG_INGEST_TOKEN>
		 - **The text above must be written exactly like that. Copy and paste and just change the token**
	 - Change "Messages to send" and "Detailed Messages to send" to **Text**
 - That's all you need in AzureDevops
	
## Configuring Dynatrace

- In Dynatrace we need 4 things: 
	- 1- Processing Rules to extract valuable data from incoming event logs
	- 2- Custom attributes to enhance the logs with extracted data
	- 3- Custom Metrics using those attributes
	- 4- Custom Dashboard so you can visualize all of it
- Luckily, we can do all of that with API to spare time **Change TENANT_ID and API_TOKEN before running the commands**

	- **1. Processing Rules**:
		- Build Event Rule
			-	  curl 'https://<TENANT_ID>.live.dynatrace.com/api/v2/settings/objects' -X  POST -H  'Accept: application/json; charset=utf-8' -H  'Content-Type: application/json; charset=utf-8' -H  'Authorization: Api-Token <SETTINGS_API_TOKEN>' -d $'[{"schemaId":"builtin:logmonitoring.log-dpp-rules","scope":"tenant","value":{"enabled":true,"ruleName":"AzureDevops_EventBuildComplete","query":"loglevel=\\"none\\" AND eventtype=\\"build.complete\\"","ProcessorDefinition":{"rule":"PARSE(content, \\"JSON{STRING:resource}(flat=true)\\")\\n| PARSE(resource,\\"JSON{STRING:reason, STRING:status, STRING:buildNumber}(flat=true)\\")\\n| FIELDS_RENAME(buildstatus:status)\\n| FIELDS_REMOVE(resource)\\n"},"RuleTesting":{"sampleLog":""}}}]'
			
	    -	Release Event Rule
		    -	  curl 'https://<TENANT_ID>.live.dynatrace.com/api/v2/settings/objects' -X POST -H 'Accept: application/json; charset=utf-8' -H 'Content-Type: application/json; charset=utf-8' -H 'Authorization: Api-Token <SETTINGS_API_TOKEN>' -d $'[{"schemaId":"builtin:logmonitoring.log-dpp-rules","scope":"tenant","value":{"enabled":true,"ruleName":"AzureDevops_EventRelease","query":"loglevel=\\"none\\" AND eventtype=\\"ms.vss-release.deployment-completed-event\\"","ProcessorDefinition":{"rule":"PARSE(content,\\"JSON:release_event\\")\\n| PARSE(release_event[\\"detailedMessage\\"][\\"text\\"], \\"\'Deployment of release \' STRING:releasenumber SPACE \'on stage \' STRING:stage SPACE STRING:result SPACE \'Time  to deploy: \' FLOAT:durationH \':\' FLOAT:durationM\':\' FLOAT:durationS\\") \\n| FIELDS_ADD(release_event[\\"resource\\"][\\"environment\\"][\\"releaseDefinition\\"][\\"name\\"] as releasename)\\n| FIELDS_ADD(release_event[\\"resource\\"][\\"project\\"][\\"name\\"] as project)\\n| FIELDS_ADD(duration: durationH *3600 + durationM * 60 + durationS)\\n| FIELDS_REMOVE(durationH, durationM, durationS, release_event)"},"RuleTesting":{"sampleLog":"{\\n \\"content\\":\\"\\"\\n}"}}}]'
		 
    -	**2. Custom Attributes**
	    -	  curl 'https://<TENANT_ID>.live.dynatrace.com/api/v2/settings/objects' -X POST -H 'Accept: application/json; charset=utf-8' -H 'Content-Type: application/json; charset=utf-8' -H 'Authorization: Api-Token <SETTINGS_API_TOKEN>' -d $'[{"schemaId":"builtin:logmonitoring.log-custom-attributes","scope":"tenant","value":{"key":"stage","aggregableAttribute":true}}, {"schemaId":"builtin:logmonitoring.log-custom-attributes","scope":"tenant","value":{"key":"result","aggregableAttribute":true}}, {"schemaId":"builtin:logmonitoring.log-custom-attributes","scope":"tenant","value":{"key":"releasenumber","aggregableAttribute":true}}, {"schemaId":"builtin:logmonitoring.log-custom-attributes","scope":"tenant","value":{"key":"releasename","aggregableAttribute":true}}, {"schemaId":"builtin:logmonitoring.log-custom-attributes","scope":"tenant","value":{"key":"reason","aggregableAttribute":true}}, {"schemaId":"builtin:logmonitoring.log-custom-attributes","scope":"tenant","value":{"key":"project","aggregableAttribute":true}}, {"schemaId":"builtin:logmonitoring.log-custom-attributes","scope":"tenant","value":{"key":"eventtype","aggregableAttribute":true}}, {"schemaId":"builtin:logmonitoring.log-custom-attributes","scope":"tenant","value":{"key":"duration","aggregableAttribute":false}}, {"schemaId":"builtin:logmonitoring.log-custom-attributes","scope":"tenant","value":{"key":"buildstatus","aggregableAttribute":true}},{"schemaId":"builtin:logmonitoring.log-custom-attributes","scope":"tenant","value":{"key":"buildnumber","aggregableAttribute":false}}]'
    - **3. Custom Metrics**
	    -	  curl 'https://<TENANT_ID>.live.dynatrace.com/api/v2/settings/objects' -X POST -H 'Accept: application/json; charset=utf-8' -H 'Content-Type: application/json; charset=utf-8' -H 'Authorization: Api-Token <SETTINGS_API_TOKEN>' -d $'[{"schemaId":"builtin:logmonitoring.schemaless-log-metric","scope":"tenant","value":{"enabled":true,"key":"log.azuredevops.builds","query":"loglevel=\\"none\\" AND eventtype=\\"build.complete\\"","measure":"OCCURRENCE","dimensions":["buildnumber","buildstatus","reason"]}},{"schemaId":"builtin:logmonitoring.schemaless-log-metric","scope":"tenant","value":{"enabled":true,"key":"log.azuredevops.release_duration","query":"loglevel=\\"none\\" AND eventtype=\\"ms.vss-release.deployment-completed-event\\"","measure":"ATTRIBUTE","measureAttribute":"duration","dimensions":["project","releasename"]}},{"schemaId":"builtin:logmonitoring.schemaless-log-metric","scope":"tenant","value":{"enabled":true,"key":"log.azuredevops.releases","query":"loglevel=\\"none\\" AND eventtype=\\"ms.vss-release.deployment-completed-event\\"","measure":"OCCURRENCE","dimensions":["project","result","stage","releasename"]}}]' 
	    
	    - **4. Dashboard**
		    - Go to Dashboards, on the left-side menu. Click import dashboard and import the .json file avaiable on this Repo.

## You're all set, enjoy Dynatrace integration

 - Keep in mind, you might need to request a Log Content Length (MaxContentLength_Bytes) increase depending on how many steps your Release Events have.
 - The integration consumes DDUs for Log Ingest and Log Metrics and will depend on how many build/release events you have. For comparison purposes, a customer with 400 events was consuming around 1 DDU per week
