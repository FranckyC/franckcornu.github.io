---
title: "Synchronize content types to a site from hub programmatically"
date: 2022-03-30T18:24:30.129Z
# post thumb
images:
  - "images/post/synchronize_content_types_from_hub/featured.jpg"
#author
author: "Franck Cornu"
# description
description: ""
# Taxonomies
categories: ["general"]
tags: [news]
type: "featured" # available type (regular or featured)
draft: false
---

The new SharePoint Online content type hub is a really geat feature to ensure a standardized information architecture across the company. Compared to the previous version, a lot of improvements has been made, especially the synchronization process with consumer sites which is now much faster and convenient.

However, to be able to add content types in a library or list coming from the hub in a specific SharePoint site, **you must first synchronize them**. Otherwise you will get a `Not found` error during the addition.

If like me, you work with the content type hub programmatically to provide site templates, here are some ways to do that depending your implementation strategy:

## Synchronize with CSOM and REST (example with a Logic App)

Unfortunately, and as far as I know, there is no [Microsoft Graph REST endpoint](https://docs.microsoft.com/en-us/graph/api/resources/contenttype?view=graph-rest-1.0) that can be used to do this synhronization for a single site. However, you can still use the `SyncContentTypesFromHubSite2` CSOM method over the `/_vti_bin/client.svc/ProcessQuery` endpoint directly with the following XML payload (example):  

```xml
<Request xmlns="http://schemas.microsoft.com/sharepoint/clientquery/2009" AddExpandoFieldTypeSuffix="true" SchemaVersion="15.0.0.0" LibraryVersion="16.0.0.0" ApplicationName=".NET Library">
   <Actions>
      <Method Name="SyncContentTypesFromHubSite2" Id="107" ObjectPathId="104">
         <Parameters>
            <Parameter Type="String">https://yourtenant.sharepoint.com/sites/yoursite</Parameter>
            <Parameter Type="Array">
               <Object Type="String">0x0101009679330D8335B9448FC3FFCD592FF7B3</Object>
               <Object Type="String">0x0101009D1CB255DA76424F860D91F20E6C411800DBFEC5E8F9D780498850457DA36850BE</Object>
            </Parameter>
         </Parameters>
      </Method>
   </Actions>
   <ObjectPaths>
      <Constructor Id="104" TypeId="{618709d0-5c34-4b0a-bd15-0406f9e62cc2}" />
   </ObjectPaths>
</Request>
```
{{< image src="images/post/synchronize_content_types_from_hub/logic_app_sync.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

From here, it is easy to build the parameters array with content type IDs to synchronize (ex: synchronize only a specific content type group in the hub):

{{< image src="images/post/synchronize_content_types_from_hub/content_type_build.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

> The XML payload shouldn't contains any spaces.

> The HTTP header `Content-Type` needs be set to `"text/xml"`.

> The call with always return an `HTTP 200` even if it does not succeed. To get the error details, you will need to parse the  response manually. 

## Use the buitlin `addContentTypesFromHub` action in a site script

If you are using site designs/templates, you can also rely on the builtin [addContentTypesFromHub](https://docs.microsoft.com/en-us/sharepoint/dev/declarative-customization/site-design-json-schema#addcontenttypesfromhub) action:

```json
{
    "$schema": "schema.json",
    "actions": [
        {
            "verb": "addContentTypesFromHub",
            "ids": ["0x01007CE30DD1206047728BAFD1C39A850120"]
        }    
    ],
    "bindata": {},
    "version": 1
}
```

## Use PnP PowerShell

Last but not least, you can use the PnP.Powershell [Add-PnPContentTypesFromContentTypeHub](https://pnp.github.io/powershell/cmdlets/Add-PnPContentTypesFromContentTypeHub.html) cmdlet. Basically it uses CSOM and `SyncContentTypesFromHubSite2` behind the scenes.

```powershell
Add-PnPContentTypesFromContentTypeHub -ContentTypes "0x010057C83E557396744783531D80144BD08D" -Site https://tenant.sharepoint.com/sites/HR
```