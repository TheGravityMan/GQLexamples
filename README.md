# ExoSense GraphQL API - Automated Asset Creation

The following tutorial will guide you through the process of claiming a Device and creating a new Asset using the ExoSense GraphQL API. Understanding of the API will enable you to take advantage of more advanced features in ExoSense, such as creating your own custom application to automate the Device claiming/Asset creation process as new Devices come online.
This guide is designed to be a simple example of a more general, higher level use case. To inquire about how to apply this information to a more specific Solution for your own needs, or have any other questions in regards to this walkthrough, please open a support ticket by emailing support@exosite.com.

This guide will Assume that you have the following pieces of information available:


|ExoSense Identifier |GraphQL Identifier|
|:-------------------|:-----------------|
|IoT Connector ID    |  `{ConnectorID}` |
|Device Identity     |  `{DeviceID}`    |
|Parent/Root Group ID|  `{RootID}`      |
|Asset Template Name |  `{TemplateName}`|


## Get Group ID and Claim Device
### Identify Target Group
The first step will be to claim your device to the desired target Group. While there are multiple ways to achieve this, we will utilize the `getGroup` guery and the root group ID to find the Target Group ID (`{TargetGroupID}`). (**NOTE:** If the target group exists further down the hierarchy, you can use `getGroup` to iterate through the group structure until you find the target group)

Alternateively, you can use the `searchForGroup` query if you know your target group's name.


**QUERY**
```
query getGroup($filters: GroupFilters) {
  groups(filters: $filters) {
    id
    name
		children{
			id
			name
		}
  }
}
```

**VARIABLES**
```
{
	"filters": {
		"group_id": "{RootID}"
	}
}
```


**RESPONSE**
```
{
	"data": {
		"groups": [
			{
				"children": [
					{
						"name": "Example Group 1",
						"id": "fji72fg8-a43b-4812-ac37-6hg678651c"
					},
					{
						"name": "Example Group 2",
						"id": "ddfg4a9-2e87-4bkj-ad2e-lk345b4070c"
					},
					{
						"name": "Example Group 3",
						"id": "df0ju6f9-b33c-4872-8011-72c9481sf732"
					}
				],
				"name": "Home",
				"id": "f67j4dec-a59f-454a-a12f-0df6184gdds8"
			}
		]
	}
}
```


### Claim Device to Target Group

The `updateDevices` mutation can be used in conjunction with the Target Group ID, IoT Connector ID, and the Device ID to claim the device into the desired group. In the `id` variable field, use the format `{ConnectorID}.{DeviceID}`. For example, if your Connector ID is `12345`, and your Device ID is `abcde`, you would input `12345.abcde`. 


**QUERY**
```
mutation updateDevices($devices: [DevicesUpdateInput]) {
	updateDevices(devices: $devices) {
		id
		__typename
	}
}
```


**VARIABLES**
```
{
	"devices": [{
		"id": "{ConnectorID}.{DeviceID}",
		"group_id": "{TargetGroupID}"
	}]
}
```


**RESPONSE**
```
{
	"data": {
		"updateDevices": [
			{
				"__typename": "Device",
				"id": "b3b2uii84fds00000.ExampleDevice"
			}
		]
	}
}
```



### IMPORTANT!
At this point, you should not proceed until the Device has a valid config_io set. To learn more about the config_io, check out our [ExoSense Data IO Schema](https://docs.exosite.io/schema/channel-signal_io_schema/) Documentation.

## Create a New Asset From a Template
The `createAssetFromT` mutation can be used to build a new Asset from a pre-defined Asset Template. However, there are 3 additional steps required before we run the mutation:

1. Identify the Template
2. Query for the Template Definition
3. Map the Channels from Template to Device

### Identify Template
One method to obtain the Template ID (`{TemplateID}`) is to use the `search` query, which allows us to search for the Template by name (`{TemplateName}`) using the `text` filter variable. Any Element (Asset, Device, Template etc..) whose name contains the `text` field input will be returned along with their `ID` and `type`.
If necessary, the `type` field can be used isolate the Template object and retrieve it's ID.

Another possible approach is to use the `Templates2` query, which will return an object containing all Templates available within your Instance.

**QUERY**
```
query search($filters: SearchFilters!) {
	search(filters: $filters) {
		results {
			id
			name
			type
		}
	}
}
```

**VARIABLES**
```
{
	"filters": {
		"text": "{TemplateName}"
	}
}
```

**RESPONSE**
```
{
	"data": {
		"search": {
			"results": [
				{
					"type": "Template",
					"name": "Example Template 1",
					"id": "o3ort5ed-345f-4142-9yh3-51543e982568"
				}
      ]
		}
	}
}
```

### Query for the Template Definition
Now we need to query for the Template's definition, which contains information necessary to complete the next step. Using the previously identified Template ID in the `id` variable field, the `template2VersionYAML` query will return the Template definition YAML. 



**QUERY**
```
query template2VersionYAML($id: ID!) {
  template2VersionYAML(id: $id) 
}
```

**VARIABLES**
```
{
	"id": "{TemplateID}"
}
```

**RESPONSE**
```
{
	"data": {
		"template2VersionYAML": {TemplateDefinition}
  }
}
```

### Map Channels from the Device to the Template
Before creating the Asset, we need to define how the channel/s from your Device are mapped to the Signal/s that will be created. Focussing on the `signals` section contained within the Template Definition will provide the necessary Signal Linkage IDs (`{SignalLinkID}`).

```
template:
  asset:
    id: a65hjuc5-db41-4304-bcdc-cd89567909
    name: Example Template 1
    description: An Example Template
    meta:
      icon: icon-asset-store
  linkages: {}
  signals:
    temperature_s:
      name: Temperature
      type: NUMBER
      units: DEG_FAHRENHEIT
      visualize: true
      root: true
      record: true
      displayFormat: leading
    alarm1_s:
      name: Alarm 1
      type: BOOLEAN
      units: null
      visualize: true
      root: true
      record: true
      displayFormat: leading
  channels:
    number1_s:
      properties:
...
```

For the example above, there are 2 Signals which will be created upon applying the Template. For each of these signals, note the Signal linkage ID :

1. `temperature_s`
2. `alarm1_s`

Channel-to-Signal mapping is defined in the variable `channelMapping[]` section of the `createAssetFromT` mutation. Each Signal will need to be mapped to a Channel from the new Device by inputing the Signal Linkage ID in the `from` field. To define the `to` field, the following elements are required:

1. IoT Connector ID
2. Device ID
3. Channel ID

Putting everything together, each entry in the `channelMapping` list is defined as follows:

```
"channelMapping": [
		{
			"from": "{SignalLinkID}",
			"to": "{connectorID}.{DeviceID}.data_in.{ChannelID}"
		}
  ]
}
```

### Create Asset From Template
Using the information we have found in the following steps, the `createAssetFromT` mutation is used to create a new Asset from the Template.


**QUERY**
```
mutation createAssetFromT($newAsset: Asset2Create,$templateId: ID,$group_id: ID!,$channelMapping: [ChannelMapping]) {
  createAssetFromT(newAsset: $newAsset, templateId: $templateId, group_id: $group_id, channelMapping: $channelMapping) {
    id
    name
		description
  }
}
```


**VARIABLES**
```
{
	"newAsset": {
		"description": "Bill's new Asset",
		"name": "Bill Rizer 1"
	},
	"templateId": "d1d255ed-394f-4142-9df3-51543e890158",
	"group_id": "18bbc821-580f-4597-b456-5e05b7432904",
	"channelMapping" : [
		{
			"from": "{SignalLinkID1}",
			"to": "{connectorID}.{DeviceID}.data_in.{ChannelID1}"
		},
    {
			"from": "{SignalLinkID2}",
			"to": "{connectorID}.{DeviceID}.data_in.{ChannelID2}"
		}
	]
}
```

**RESPONSE**
```
{
	"data": {
		"createAssetFromT": {
			"description": "Bill's new Asset",
			"name": "Bill Rizer 1",
			"id": "54720dae-5df8-4337-ade0-93e2cab52f6c"
		}
	}
}
```


