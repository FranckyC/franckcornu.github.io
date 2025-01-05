---
title: "Leverage PnP Modern Search Core components and Microsoft Search in your Docusaurus documentation"
date: 2025-01-03
# post thumb
images:
 - images/post/integrate-components-with-docusaurus/docusaurus.png
#author
author: "Franck Cornu"
# description
description: "Detail how to integrate PnP Modern Search Core components and Microsoft Search with Docusaurus sites"
# Taxonomies
categories: ["docusaurus","Microsoft Search"]
tags: [post]
type: "featured" # available type (regular or featured)
draft: false
sidebar: right
index: true
showimage: false
_build:
  list: true
--- 
In this blog post, I show you how to integrate the [PnP Modern Search Core components](https://github.com/microsoft-search/pnp-modern-search-core-components) into your enterprise Docusaurus documentation sites to add search capability. 

> The GitHub solution associated to this post is available here: [https://github.com/microsoft-search/pnp-modern-search-core-components/tree/main/samples/docusaurus](https://github.com/microsoft-search/pnp-modern-search-core-components/tree/main/samples/docusaurus) 

## What is Docusaurus?

Docusaurus ([https://docusaurus.io/](https://docusaurus.io/)) is a static site generator created by Facebook and implemented with React. It provides an easy way for dev teams to write their documentation using Markdown syntax.
I personnaly use it for all my projects (interna/external) as it contains a lot of features and is very easy to use and customize. 

If you don't know about this tool, I strongly encourage you to take a look.

## Limitation with search

By default, Docusaurus doesn't come with a built-in search engine for your website. It is up to you to configure one. For public websites with no authentication, it does provides a straightforward integration with [Algolia](https://docusaurus.io/docs/search#using-algolia-docsearch).
For example, this what I use to use for the [public documentation](https://microsoft-search.github.io/pnp-modern-search-core-components/) of the PnP Modern Search Components and it works quite well.

**The thing is, in a enterprise scenario, if you decided to use Docusaurus to write your internal documentation, you won't likely expose it publicly to be indexed by a third party tool just to get a search capability**.

For only few sites in your organization, you might say search is not that important if navigation is well managed. Fair enough. But when you have several sites made by different teams having their own structure with a lot of content, the search feature becomes essential.

In the next sections, I'll detail how to use Microsoft Search to index an internal documentation site and provide a search experience directly in it using PnP search components. 

## Prepare your site for Microsoft Search

### Configure meta-tags

Having search capabilities often means having relevant filters for your content. Regarding websites, this can be achieved by leveraging HTML meta-tags.

In Docusaurus, meta-tags can be set directly in the [docusaurus.config.ts](https://github.com/microsoft-search/pnp-modern-search-core-components/blob/main/samples/docusaurus/docusaurus.config.ts) file:

```javascript
themeConfig: {
	...
	metadata: [
	  {name: 'projectName', property: "projectName", content: 'PnP Modern Search Core - Docusaurus Demo'},
	  {name: 'projectType', property: "projectType", content: 'Public'},
	  {name: 'projectTechnologies', property: "projectTechnologies", content: 'Docusaurus,TailwindCSS,Azure DevOps,Microsoft Search'},
	  {name: 'projectDescription', property: "projectDescription", content: 'Integration demo of PnP Modern Search core components in a Docusaurus site'},
	  {name: 'projectRepository', property: "projectRepository", content: 'https://github.com/microsoft-search/pnp-modern-search-core-components'},
	  {name: 'projectContact', property: "projectContact", content: 'Franck Cornu'},
	  {name: 'source', property: "source", content: 'Documentation'},
	],
	...
} satisfies Preset.ThemeConfig
```

When configured here, meta-tags are available for **all the pages in your site**:

{{< image src="/images/post/integrate-components-with-docusaurus/meta-tags.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

This configuration is required to make sure all the content is tagged correctly. If needed, you can also set meta-tags at [page level](https://docusaurus.io/docs/seo#single-page-metadata).

These meta tags will be then mapped to your search schema in Microsoft Search when creating the connector.

### Deploy your site

In my context, I use an Azure App Service (Windows, NodeJS 20+) to host the documentation. Docusaurus being a static site generator, it technically doesn't require a web server and can be deloyed to an [Azure Static Web App](https://learn.microsoft.com/en-us/azure/static-web-apps/overview) or an [Azure Blob container with the static webiste feature enabled](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website) (in this case, you can't add authentication on top like an App Service or a Static Web Site provide with EasyAuth).
When deploying a Docusaurus site to an App Service, you'll need to write a dummy [server](https://github.com/microsoft-search/pnp-modern-search-core-components/blob/main/samples/docusaurus/src/server/server.ts) and configure a [web.config](https://github.com/microsoft-search/pnp-modern-search-core-components/blob/main/samples/docusaurus/src/server/web.config) rewrite rule to serve static assets from Docusaurus build folder. In my case I use a basic Node.js Express server: 

```javascript
import express from "express";

const app = express();
const port = process.env.port || process.env.PORT || 3333;

// 'build' is the folder where the static contents are generated by Docusaurus
app.use(express.static(__dirname + '/build'));

app.listen(port, () => {
  console.log(`Server started on Azure on port ${port}`)
});
```

Also, for convenience, and to avoid uploading the whole `node_modules`, I'm using [Webpack](https://github.com/microsoft-search/pnp-modern-search-core-components/blob/main/samples/docusaurus/src/server/webpack.server.js) to get a standalone bundle file to be uploaded:

```xml
...
<rewrite>
  <rules>
	...
	<!-- All other URLs are mapped to the node.js site entry point -->
	<rule name="DynamicContent">
	  <conditions>
		<add input="{REQUEST_FILENAME}" matchType="IsFile" negate="True"/>
	  </conditions>
	  <action type="Rewrite" url="server.js"/>
	</rule>
  </rules>
</rewrite>
...
```

In the end, once deployed to the App Service, my application looks like this:

{{< image src="/images/post/integrate-components-with-docusaurus/app_service_wwwroot.png" caption="" alt="alter-text" position="center" class="img-fluid" title="wwwroot" webp="false" >}}

### Configure authentication

The next step is to configure autentication for your deployed web application. In an enterprise scenario, you likely want your site to be protected by Entra ID. For this part, you have basically two options:

- Implement the authentication part directly in your Docusaurus site, [for instance using MSAL.js](https://dev.to/ib1/docusaurus-authentication-with-entra-id-and-msal-417b) (Kudos to my fellow coworker @Igor Bertnyk!). The issue with that approach is your site content will become inaccessible to the search crawler. The crawler being a daemon process with no user interaction and MSAL.js being purely client-side, when it will hit your page, it will just see the sign-in page and will you will end-up with only 1 page indexed as it can't authenticate.

- Use the _"[EasyAuth](https://learn.microsoft.com/en-us/azure/app-service/overview-authentication-authorization)"_ feature of App Service. With this type of authentication, your content will remain accessible to the crawler, as the authentication will be performed before accessing to your site. More importantly, the Microsoft Search Graph connector **[Enterprise Websites ](https://learn.microsoft.com/en-us/microsoftsearch/enterprise-web-connector)** connectors (Cloud or On-Premises) support this scenario and will be able to connect to your site this way.

{{< image src="/images/post/integrate-components-with-docusaurus/easyauth.png" caption="" alt="alter-text" position="center" class="img-fluid" title="wwwroot" webp="false" >}}

### Create connector in Microsoft Search

In enterprise search scenario, you must use the Enterprise Websites "On-Premise" connector as your webiste won't be available over the public internet, and therefore accessible by the Microsoft cloud connector.

In this case, you need first to setup and register a machine with the [Graph Connector Agent](https://learn.microsoft.com/en-us/microsoftsearch/graph-connector-agent) that has access to this URL (most of the time, a virtual machine inside the company internal network boundary).

In the Microsoft Search admin center, create a new connector, setting the information for your website. For the authentication part, use the application ID for both resource and clients ID fields and use a generated client secret from the Entra ID application used by EasyAuth:

{{< image src="/images/post/integrate-components-with-docusaurus/enterprise_connector.png" caption="" alt="alter-text" position="center" class="img-fluid" title="wwwroot" webp="false" >}}

> The website needs to be hosted in the same tenant as the Microsoft 365 one. **You can't index an internal website protected by Entra ID hosted in a different tenant**

Once the connection is authorized, on the _"Content"_ tab, setup your search schema mapping the meta-tags you defined earlier:

{{< image src="/images/post/integrate-components-with-docusaurus/add-meta-tags.png" caption="" alt="alter-text" position="center" class="img-fluid" title="wwwroot" webp="false" >}}

### Integrate components into your site

Once the connector is ready, we can now integrate PnP Modern Search components into the website to get results.

In the package.json, first add the  `@pnp/modern-search-core` [dependency](https://www.npmjs.com/package/@pnp/modern-search-core):

```javascript
...
"dependencies": {
	..
    "@pnp/modern-search-core": "1.1.1"
	...
}
```

#### Configure the components with authentication and deal with SSR

Because Docusaurus uses React SSR (Server Side Rendering), you can't import components directly using regular `import { <...> } from '@pnp/modern-search-core'` statements on top of your files as this will fail the build process. PnP Modern Search components are web components meant to be executed in the browser and can't be packaged for server side rendering (for instance the 'window' object is not accesssible from there). As a workaround, dependencies are loaded asynchronously, when the actual page is rendered in the browser (see [https://docusaurus.io/docs/advanced/ssg](https://docusaurus.io/docs/advanced/ssg) for more info about SSR workarounds):

We use the [swizlling](https://docusaurus.io/docs/swizzling) technique to override the root component [theme/Root.tsx](https://github.com/microsoft-search/pnp-modern-search-core-components/blob/main/samples/docusaurus/src/theme/Root.tsx) (rendered once in this application), to load the components asynchronously. This way, components can be used on all pages:

```typescript
import { useEffect} from "react";

export default function Root({children}) {

    useEffect(() => {

        const initialize = async () => {
            
            await import('@pnp/modern-search-core/dist/es6/exports/define/pnp-search-results');
            await import('@pnp/modern-search-core/dist/es6/exports/define/pnp-search-input');
            await import('@pnp/modern-search-core/dist/es6/exports/define/pnp-search-filters');
            await import('@pnp/modern-search-core/dist/es6/exports/define/pnp-search-verticals');
    
            const { LoginType, Providers, SimpleProvider, ProviderState, Msal2Provider,TemplateHelper, registerMgtLoginComponent } = await import('@pnp/modern-search-core');
    
            const login = async () => {

                // Easy auth scenario
                try {
                    await fetch('/.auth/login');  
                } catch (error) {
                    console.error('Error while login', error);
                    return null;
                }
            }

            const logout = async () => {

                // Easy auth scenario
                try {
                    await fetch('/.auth/logout');  
        
                } catch (error) {
                    console.error('Error while logout', error);
                    return null;
                }
            }
                
            const getAuthInfo = async () => {

                // Easy auth scenario
                try {
                    const response = await fetch('/.auth/me');  
                    const payload = await response.json();  
                    return payload && payload[0];
                } catch (error) {
                    console.error('Error while getting access token', error);
                    return null;
                }
            };

            const refreshAccessToken = async () => {

                // Easy auth scenario
                try {
                    const response = await fetch('/.auth/refresh');  
                    const payload = await response.json();  
                    return payload && payload[0];
                } catch (error) {
                    console.error('Error while getting access token', error);
                    return null;
                }
            };

            const getAccessToken = async () => {

                const authInfo = await getAuthInfo();
                if (new Date(authInfo.expires_on) >= new Date()){
                    const refreshToken = await refreshAccessToken();

                    return refreshToken;
                }
                return authInfo?.access_token;
            };

            const authInfo = await getAuthInfo();

            if (authInfo) {
                const simpleProvider = new SimpleProvider(getAccessToken, login, logout);
               
                simpleProvider.getActiveAccount = () => {
                    return {
                        id: authInfo.user_id
                    }
                }

                Providers.globalProvider = simpleProvider;
                Providers.globalProvider.setState(ProviderState.SignedIn)
            
            } else {

                Providers.globalProvider = new Msal2Provider({
                    clientId: process.env.ENV_MSSearchAppClientId,
                    authority: `https://login.microsoftonline.com/${process.env.ENV_MSSearchAppTenantId}`,
                    domainHint: process.env.ENV_MSSearchAppDomain,
                    redirectUri: `${process.env.ENV_SiteUrl}`,
                    loginType: LoginType.Popup
                });
            }
    
            // To avoid conflicts with MDX
            TemplateHelper.setBindingSyntax('[[', ']]');
    
            registerMgtLoginComponent();
        }

        initialize();
    });

    return children;
}
```

For authentication, we create a dummy Microsoft Graph Toolkit provider, getting access token directly from EasyAuth builtin [endpoints](https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-oauth-tokens#retrieve-tokens-in-app-code) (`/.auth/me|refresh|login|logout`). 

The next step is now to provide a basic search experience, with a search box always accessible on top of pages and a dedicated search page to browse results.

#### Adding the searchbox

Docusaurus allows you to override the [default search bar](https://docusaurus.io/docs/search#using-your-own-search) rendering by providing your own component. Again, we use Docusaurus swizzling technique by creating the `theme/SearchBar.tsx` file and adding the PnP Search box components. We also add the Microsoft Graph Tooolkit login component to indicate if the user is logged in or not (with the possibility to logout if needed):

```typescript
import { wrapWc } from 'wc-react';
import { MgtLogin, SearchInputComponent } from '@pnp/modern-search-core';
import { PageOpenBehavior, QueryPathBehavior } from '@pnp/modern-search-core/dist/es6/helpers/UrlHelper';
import React from "react";
import BrowserOnly from '@docusaurus/BrowserOnly';

const SearchInputWebComponent = wrapWc<Partial<SearchInputComponent>>('pnp-search-input');
const MgtLoginWebComponent = wrapWc<Partial<MgtLogin>>('mgt-login');

export default function SearchBar() {

    return  <div className='flex items-center space-x-3'>
                <BrowserOnly>
                {() => {
                    return  <>
                                <SearchInputWebComponent 
                                    id='f566fb98-4010-485b-a44b-b53c16852cd6'
                                    inputPlaceholder='Enter a keyword...'
                                    searchInNewPage={true}
                                    pageUrl={`${new URL(`${process.env.ENV_SiteUrl}/search`)}`}
                                    queryPathBehavior={QueryPathBehavior.QueryParameter}
                                    queryStringParameter='k'
                                    defaultQueryStringParameter='k'
                                    openBehavior={PageOpenBehavior.Self}
                                />

                                <MgtLoginWebComponent loginView='avatar'></MgtLoginWebComponent>
                            </>;
                }}   
                </BrowserOnly>                
            </div>;
}
```
Few things to notice here:
- We consume PnP Modern Search components as React components there thanks to the [wc-react](https://www.npmjs.com/package/wc-react) utility library. This allows to easily set component properties:

```typescript
const SearchInputWebComponent = wrapWc<Partial<SearchInputComponent>>('pnp-search-input');
```
- Because Docusaurus uses React SSR (Server Side Rendering), it is important to enclose the component with the `<BrowserOnly>` [wrapper](https://docusaurus.io/docs/advanced/ssg#browseronly). This basically excludes the components from server compilation.

- We use [Webpack EnvironmentPlugin](https://github.com/microsoft-search/pnp-modern-search-core-components/blob/18ff3c7d2435ba9bdda674d5d607e045e984eba9/samples/docusaurus/docusaurus.config.ts#L189) to dynamically set the search page URL using environment variables at bundling time.

#### Creating the search page

Last step of the process, we create the dedicated search page search.tsx using again, React components enclosed with `<BrowserOnly>` wrapper:

```typescript
import Layout from "@theme/Layout";
import { wrapWc } from "wc-react";
import { SearchFiltersComponent, SearchInputComponent, SearchResultsComponent } from "@pnp/modern-search-core";
import { BuiltinFilterTemplates } from "@pnp/modern-search-core/dist/es6/models/common/BuiltinTemplate";
import { FilterConditionOperator } from "@pnp/modern-search-core/dist/es6/models/common/IDataFilter";
import { FilterSortDirection, FilterSortType } from "@pnp/modern-search-core/dist/es6/models/common/IDataFilterConfiguration";
import { EntityType } from "@pnp/modern-search-core/dist/es6/models/search/IMicrosoftSearchRequest";
import BrowserOnly from "@docusaurus/BrowserOnly";

const SearchInputWebComponent = wrapWc<Partial<SearchInputComponent>>('pnp-search-input');
const SearchFiltersWebComponent = wrapWc<Partial<SearchFiltersComponent>>('pnp-search-filters');
const SearchResultsWebComponent = wrapWc<Partial<SearchResultsComponent>>('pnp-search-results');

export default function Searchpage(): JSX.Element {
    
    return  <Layout>  
                <main className="container container--fluid margin-vert--lg">
                    <BrowserOnly>
                    {() => {
                        return  <>
                                    <SearchInputWebComponent 
                                        id='edfdea93-23c9-4ba6-94d9-a848a1384104'
                                        inputPlaceholder='Enter a keyword...'
                                        defaultQueryStringParameter={"k"}
                                    ></SearchInputWebComponent>
                    
                                    <SearchFiltersWebComponent
                                        id="4f5ad5cd-8626-40ab-9c89-466a64f57a8e"
                                        filterConfiguration={[
                                            {
                                                displayName: "Type",
                                                filterName:"filetype",
                                                template: BuiltinFilterTemplates.CheckBox,
                                                isMulti:false, maxBuckets:50, 
                                                operator: FilterConditionOperator.OR,
                                                showCount: false,
                                                sortBy: FilterSortType.ByCount,
                                                sortDirection: FilterSortDirection.Ascending,
                                                sortIdx:1
                                            },
                                            {
                                                displayName: "Created",
                                                filterName:"createddatetime",
                                                template: BuiltinFilterTemplates.CheckBox,
                                                isMulti:false, maxBuckets:50, 
                                                operator: FilterConditionOperator.OR,
                                                showCount: false,
                                                sortBy: FilterSortType.ByCount,
                                                sortDirection: FilterSortDirection.Ascending,
                                                sortIdx:1
                                        }]}
                                        searchResultsComponentIds={["97c0a5dd-653b-4b41-8ced-7ed0feb7da88","68f37193-8a2e-4c09-8337-1c3db978ccb8"]}
                                    ></SearchFiltersWebComponent>  
                                                    
                                    <SearchResultsWebComponent
                                        id="68f37193-8a2e-4c09-8337-1c3db978ccb8"
                                        defaultQueryText="*"
                                        queryTemplate="{searchTerms}"
                                        selectedFields={["title","weburl","filetype","source","keywords","url","iconUrl","createddatetime"]}
                                        searchInputComponentId="edfdea93-23c9-4ba6-94d9-a848a1384104"
                                        searchFiltersComponentId="4f5ad5cd-8626-40ab-9c89-466a64f57a8e"
                                        connectionIds={[`/external/connections/${process.env.ENV_MSSearchConnectionId}`]}
                                        entityTypes={[EntityType.ExternalItem]}
                                        showDebugData={true}
                                        pageSize={10}
                                    >
                    
                                        <template data-type="items">
                                            <div data-for='item in items'>
                                                    <div className="mt-4 mb-4 ml-1 mr-1 grid gap-2 grid-cols-searchResult">
                                                    <div className="h-7 w-7 text-textColor m-1">
                                                        <img src="[[ item.iconurl ]]"/>
                                                    </div>
                                                    <div className="dark:text-textColorDark">
                                                        <div className="font-semibold mt-1 mb-1 ml-0 mr-0 ">
                                                            <a href="[[item.url]]" target="_blank">[[ item.title]]</a>
                                                        </div>
                                                        <div className="mt-1 mb-1 ml-0 mr-0 line-clamp-4" data-html>
                                                            [[item.summary]]
                                                        </div>
                                                    </div>
                                                    </div>          
                                            </div>
                                        </template>
                    
                                    </SearchResultsWebComponent>
                                </>;
                        
                    }}
                    </BrowserOnly> 
                </main>
            </Layout> 
}
```

To consume results from the Microsoft Graph Connector defined ealrier, we specify the connection ID in the `SearchResultsWebComponent` component like this:

```typescript
connectionIds={[`/external/connections/${process.env.ENV_MSSearchConnectionId}`]}
```

The rest of the configuration is done according components [documentation and properties](https://azpnpmodernsearchcoresto.z9.web.core.windows.net/latest/index.html?path=/docs/introduction-getting-started--docs).

Et voilà! Now when the website is accessed, users will be logged automatically via EasyAuth and will be able to search for website content site using components and Microsoft Search:

{{< image src="/images/post/integrate-components-with-docusaurus/homepage.png" caption="" alt="alter-text" position="center" class="img-fluid" title="wwwroot" webp="false" >}}

You can see the full working example here: [https://github.com/microsoft-search/pnp-modern-search-core-components/tree/main/samples/docusaurus](https://github.com/microsoft-search/pnp-modern-search-core-components/tree/main/samples/docusaurus)