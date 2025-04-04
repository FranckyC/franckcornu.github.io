---
title: "Build a company wide documentation platform with Azure Application Gateway and Microsoft Search"
date: 2025-04-01
# post thumb
images: 
- images/post/manage-org-documentation-app-gateway/featured.jpeg
#author
author: "Franck Cornu"
# description
description: "Detail how to merge internal documentation sites into an Azure Application Gateway as an unique endpoint to be crawled by Microsoft Search"
# Taxonomies
categories: ["Azure Application Gateway","App Service","Microsoft Search"]
tags: [post]
type: "featured" # available type (regular or featured)
draft: false
sidebar: right
index: true
showimage: false
_build:
  list: true
--- 

Managing technical knowledge within a company, especially in large organizations with numerous teams, can be incredibly challenging. Often, project or technical content that could benefit various teams is scattered across multiple locations and presented in diverse formats. This dispersion makes the information difficult to access, hard to navigate, and, in some cases, completely hidden from others.

I recently worked on an initiative to transition from heterogeneous documentation formats used by different internal technical teams (such as Word documents and wikis) to a "documentation-as-code" approach leveraging [Docusaurus](https://docusaurus.io/).

In this blog post, I outline the method we took to establish a centralized endpoint for users to access technical content across the company and ensure all this content is easily accessible and searchable using Microsoft Search. To demonstrate the concept, I use an example involving multiple web applications used for content publication, in that case documentation sites implemented with Docusaurus.

{{< image src="/images/post/manage-org-documentation-app-gateway/without_gateway.png" caption="" alt="Without Gateway" position="center" class="img-fluid" title="image title" webp="false" >}}

The goal is to go from an architecture where multiple sites are spread across the organization to an architecture with only one entry point using the following building blocks:

- **Azure Application Gateway**: to provide unique URL to users and route path-based requests to the right underlying web application transparently to users.
- **App Services**: to host the documentation content. Applications can be anonymous or protected by Entra ID (using EasyAuth).
- **Microsoft Search and Enterprise Website Graph Connector**: to crawl the content from all the web applications under the entry point.

{{< image src="/images/post/manage-org-documentation-app-gateway/with_gateway.png" caption="" alt="Target architecture" position="center" class="img-fluid" title="image title" webp="false" >}}

This architecture has several benefits:

- It makes easier for user to access content with only one URL providing a central search capability from a root site (for instance using the [PnP Modern Search Core Components](https://github.com/microsoft-search/pnp-modern-search-core-components)).
- It bypasses the Microsoft Search "Enterprise Websites" graph connector limitations allowing only 50 URLs to be crawled per connection and preventing adding new URLs after the initial creation (when a new site needs to be indexed, it can't be added to the connector URL list. You have to delete and recreate the connection from scratch. Not ideal). With this approach,you only have to configure the connector once. New sites added under the main entry point will be picked-up automatically by the crawler.
- It supports both anonymous and Entra ID secured sites scenarios.

Now we detailled the architectyre, let's see how to implement it.

## Configure Application Gateway

### 1. Configure backend pools

The first step is to configure the backend pools for all the app services we want to aggregate (root web application included):

{{< image src="/images/post/manage-org-documentation-app-gateway/backendpools.png" caption="" alt="Backend pools" position="center" class="img-fluid" title="image title" webp="false" >}}

### 2. Configure health probes

A custom health probe is needed for app services using authentication. Health probes don't handle authentication flow by default so we need to ensure HTTP 401/403 responses are valid responses for a backend pools. This probe covers both anynomous/authenticated scenarios.

<div style="display:flex">
  <div>
    <ul>
      <li>Name: <strong>azurewebsitesprob</strong></li>
      <li>Protocol: <strong>HTTPS</strong></li>
      <li>Pick host name from backend settings: <strong>Yes</strong></li>
      <li>Pick port from backend settings: <strong>Yes</strong></li>
      <li>Path: <strong>/</strong></li>
      <li>Interval (seconds): <strong>30</strong></li>
      <li>Timeout (seconds): <strong>30</strong>. Depending on your app service settings (e.g., 'Always on' or not), you may increase that value. For instance, if the app is not able to respond in that timeframe, the gateway will throw an HTTP 502 error to users.</li>
      <li>Unhealthy threshold: <strong>3</strong></li>
      <li>Use probe matching conditions: <strong>Yes</strong></li>
      <li>HTTP response status code match: <strong>200-403</strong></li>
      <li>HTTP response body match: (Empty)</li>
      <li>Backend settings: (Assigned later)</li>
    </ul>
  </div>
  
  {{< image src="/images/post/manage-org-documentation-app-gateway/healthprobe.png" caption="" alt="Health probe" position="center" class="img-fluid" title="image title" webp="false" >}}
</div>

### 3. Configure backend settings

Then create a backend settings used by **all** applications with the following configuration:

<div style="display:flex">
  <div>
    <ul>
      <li>Backend protocol: <strong>HTTPS</strong></li>
      <li>Backend port: <strong>443</strong></li>
      <li>Backend server's certificate is issued by a well-known CA: <strong>Yes</strong> (the one from <code>azurewebsites.net</code> domain)</li>
      <li>Cookie-based affinity: <strong>Disable</strong></li>
      <li>Connection draining: <strong>Disable</strong></li>
      <li>Request time-out (seconds): <strong>30</strong></li>
      <li>Override backend path: This one is pretty important. Leaving the default means the targeted app service must be able to resolve the path-based URL route from the gateway (e.g., https://app1internal.azurewebsites.net/app1.)</li>
      <li>Override with new host name: <strong>No</strong>. </li>
      <li>Host name override: <strong>Pick host name from backend target</strong>. This will fallback to the <code>https://{site}.azurewebsites.net</code> app service default URL.</li>
      <li>Use custom probe: <strong>Yes</strong></li>
      <li>Custom probe: <strong>azurewebsitesprob</strong>. The probe you created earlier</li>
    </ul>
  </div>
  
  {{< image src="/images/post/manage-org-documentation-app-gateway/backendsettings.png" caption="" alt="Backend settings" position="center" class="img-fluid" title="image title" webp="false" >}}
</div>

### 4. Configure listeners

Create a new listener to use when users go to the `docs.mycompany.com` application with the following parameters:

<div style="display:flex">
  <div>
   <ul>
    <li>Listener name: <strong>https</strong></li>
    <li>Frontend IP: <strong>Private</strong>. In a business scenario, this should be a private IP to ensure the application isn't accessible from the public internet.</li>
    <li>Protocol: <strong>HTTPS</strong></li>
    <li>Port: <strong>443</strong></li>
    <li>Certificate: <strong>Create new</strong>. Upload a .pfx certificate here for the domain <code>docs.mycompany.com</code> and use the certificate password. This certificate can also be uploaded to a Key Vault first and used here.</li>
    <li>Listener type: <strong>Multi site</strong></li>
    <li>Host type: <strong>Single</strong> (as the app is intended to be accessed from a unique URL)</li>
    <li>Host name: <code>docs.mycompany.com</code></li>
  </ul>
  </div>
  
  {{< image src="/images/post/manage-org-documentation-app-gateway/listener.png" caption="" alt="Listener" position="center" class="img-fluid" title="image title" webp="false" >}}
</div>

Unlike App Services, Azure application gateway does not require to do a domain validation so you can use an internal domain here.

### 5. Configure routing rules

Finally, put all pieces together in a routing rule with the following parameters using the listener you created:

{{< image src="/images/post/manage-org-documentation-app-gateway/rules.png" caption="" alt="Rules" position="center" class="img-fluid" title="image title" webp="false" >}}

<div style="display:flex">
  <div>
   <ul>
    <li>Priority: <strong>1</strong></li>
    <li>Listener: <strong>https</strong></li>
    <li><strong>Backend targets</strong>
      <ul>
        <li>Target type: <strong>Backend pool</strong></li>
        <li>Backend target: <strong>rootportalbackend</strong></li>
        <li>Backend settings: <strong>backendsettings</strong></li>
      </ul>
    </li>
    <li><strong>Path-based  (do this for all all applications)</strong>
      <ul>
        <li>Path: <strong>/app1*</strong>. The '*' character is quite important here and will allow anything after the path to be routed as well.</li>
        <li>Target name: <strong>app1</strong></li>
        <li>Backend setting name: <strong>backendsettings</strong></li>
        <li>Backend pool: <strong>app1portalbackend</strong></li>
      </ul>
    </li>
  </ul>
  </div>
  
</div>

The next step is to configure app services.

## Configure App Services

## Auth settings configuration

The technology used to publish your content within the app doesn't really matter here. However, the way the app service is configured will be quite important to function properly behind the gateway.

### Without authentication

If the app service does not use any authentication (e.g Entra ID), the configuration will be as follow (Bicep): 

```yaml
param webAppName string

resource webApp 'Microsoft.Web/sites@2020-06-01' existing = {
  name: webAppName
}

resource webAppConfig 'Microsoft.Web/sites/config@2022-09-01' = {
  name: 'authsettingsV2'
  kind: 'string'
  parent: webApp
  properties: {
    httpSettings: {
      requireHttps: true
      forwardProxy: {
        convention: 'Custom'
        customHostHeaderName: 'X-Original-Host'
      }
    }
  }
}

```

- `forwardProxy`: this setting will ensure the app service will handle the hostname correctly behind the application gateway reverse proxy. Typically, it ensures the hostname from the orignal query is kept from the original request without exposing the default app service host name (ie. `azurewebsites.net`) to the client.
- `customHostHeaderName`: By default this header is sent by the application gateway.

### With Authentication

However, if the application uses Entra ID authentication (ex: root app `'/'` and `'/app1'` in the example above), you'll need additional settings to make the OAuth2 redirection flows to work properly with path-based routes:

```yaml
param webAppName string
param webAppBaseUrl string
param webAppHostUrl string

resource webApp 'Microsoft.Web/sites@2020-06-01' existing = {
  name: webAppName
}

resource webAppConfig 'Microsoft.Web/sites/config@2022-09-01' = {
  name: 'authsettingsV2'
  kind: 'string'
  parent: webApp
  properties: {
    httpSettings: {
      requireHttps: true
      routes: {
          apiPrefix: '/${webAppBaseUrl}/.auth'
      }
      forwardProxy: {
        convention: 'Custom'
        customHostHeaderName: 'X-Original-Host'
      }
    }
    login: {
      allowedExternalRedirectUrls: [
        '${webAppHostUrl}/${webAppBaseUrl}/.auth/login/aad/callback'
      ]
      tokenStore: {
          enabled: true
      }
    }
  }
}

```

- `routes/apiPrefix` This setting will ensure the EasyAuth Redirect URL with be crrectly handled. **If not present, the redirect URL won't work and authentication will fail**. Huge thanks to this blog [post](https://techcommunity.microsoft.com/blog/azurenetworkingblog/understanding-azure-app-service-authentication-challenges-with-path-based-routin/4372730)

- `login/allowedExternalRedirectUrls`: make sure the redirect URL is allowed. **You'll have to configure this redirect URL in your SPN (Web Platform)**.

- `tokenStore`: allow the application code to retrieve access token once authenticated.

## Application routes configuration

Because the application gateway backend setting _"Override backend path"_ is set to empty, it means applications need to handle the path-based routing correctly (ex: `/app1`). Here is an example how to configure these routes for the _"app1"_ application using a Node.js Express server and Docusaurus (you can refer to [this sample](https://github.com/microsoft-search/pnp-modern-search-core-components/tree/main/samples/docusaurus) to see the base configuration):

```javascript
import express from "express";

const app = express();
const port = process.env.port || process.env.PORT || 3333;

// A default route should exist (even if not functional) to make application gateway probes working (i.e. HTTP 200) (in the case of the site doesn't have any authentication).
// With EasyAuth, the probe will hit the authentication endpoint first with a HTTP 401.
app.use('/', express.static(__dirname + '/build'));

// Allow the app to correctly handle requests from the gateway
app.use("/app1", express.static(__dirname + '/build'));

app.listen(port, () => {
  console.log(`Server started on Azure on port ${port}`)
});
```
Also, if you are using Docusaurus, you'll also need to update the `baseUrl` to point to that path (`docusaurus.config.ts`):

```typescript
const config: Config = {
  ...

  // Set the /<baseUrl>/ pathname under which your site is served

  baseUrl: '/app1',
```

# Microsoft search configuration

Last but not least, the Microsoft Search configuration. Whether your site is accessible externally or not, you'll need a different type of Enterprise Websites Graph Connector. For publicly available websites, you can use the [Enterprise Websites Microsoft Graph Connector (Cloud)](https://learn.microsoft.com/en-us/microsoftsearch/enterprise-web-connector). However, for internal sites, you'll need to use the [Enterprise Websites Microsoft Graph Connector (On-Premises)](https://learn.microsoft.com/en-us/microsoftsearch/enterprise-web-connector-onprem) with a dedicated Graph Connector Agent that has access to your company's network.

Also, for all the sites to be crawled, **you need to explicitly reference their URLs as hyperlinks in the root site (for example in the top navigation) or add them to the sitemap**. Otherwise, the web crawler won't be able to see them.

Finally the connection creation is pretty straighforward:

{{< image src="/images/post/manage-org-documentation-app-gateway/microsoft_search.png" caption="" alt="Microsoft Search" position="center" class="img-fluid" title="image title" webp="false" >}}

**If you have multiple web applications using authentication under the main gateway entry point, as long as you keep the same SPN, the Microsoft Search web crawler will be able to access the content (as it is the same resource). If for any reason, you need to secure a web application with a different SPN (ex: to restrict on a specific Entra ID group), you'll need to setup a new connection for that site.**

With this setup, it is now possible to search content from all the sites aggreagated under the gateway. The connector can be used to create custom search experiences or even be used as a knowledge source for a Copilot agent!

To know how to create a search interface for you users from a custom Docusaurus site and the PnP Modern Search Core Components, you can refer to my previous other blog post [Leverage PnP Modern Search Core components and Microsoft Search in your Docusaurus documentation](https://blog.franckcornu.com/post/integrate-components-with-docusaurus/).

Enjoy!
