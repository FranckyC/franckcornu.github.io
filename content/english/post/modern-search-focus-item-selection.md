---
title: "PnP Modern Search: focus on item selection feature"
date: 2022-02-13T00:00:00.000Z
# post thumb
images:
  - "images/post/modern-search-focus-item-selection/featured.png"
#author
author: "Franck Cornu"
# description
description: ""
# Taxonomies
categories: ["modern-search"]
tags: [post]
type: "featured" # available type (regular or featured)
draft: false
sidebar: right
showimage: false
---

In the latest release of PnP Modern Search (`4.5.3`), we've added the ability to connect two Search Results Web Parts together. It went quite unnoticed at the time but it actually unlocks a lot of cool new scenarios requiring dynamic filtering, for instance, to build dynamic dashboards experiences. In this article I will review how this works and how you can benefit from this feature for your search pages.

## How to connect two search results Web Parts together?

### Prerequisite: understand how dynamic filering works

When you connect two Web Parts together, you basically match a value from a selected source item property against a value contained in a property target items. In other terms, when an item is selected:

> "Retrieve all items where the property **X** contains the value of the property **Y** of the selected item"

Notice that properties don't need to be necessarily the same in both sides as two different properties can share common values. You can already experience this configuration with the dynamic filtering feature for **List** and **Library** default Web Parts:

{{< image src="images/post/modern-search-focus-item-selection/ootb_dynamic_filtering.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

The concept is the same with PnP Search Results Web Parts, except you get more flexibility on how the query is made.

### Example with a taxonomy column

To illustrate the item selection feature between two Search Results Web Parts, I take the example of a devices catalog that should display the related documentation when the user select one or more device from the list:

{{< image src="images/post/modern-search-focus-item-selection/item_selection_demo.gif" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

To make the link between items, the two lists share a same managed metadata column named "_Related product_":

{{< image src="images/post/modern-search-focus-item-selection/lists_setup.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

Beacuse, we use SharePoint search, I also created a corresponding managed property in the search schema using the term ID property (not the text one):

{{< image src="images/post/modern-search-focus-item-selection/search_schema.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

The Search Results Web Parts connection configuration is always done in **two** steps.

#### 1- Allow item selection in source Web Part

The first step is to configure a source Web Part where items will be selected and then enable the item selection feature in the layout configuration page under _Common_ options:

{{< image src="images/post/modern-search-focus-item-selection/step1.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

> Item selection is available for all layouts.

You can also turn on/off multiple selection for results.

#### 2- Configure connection on target Web Part(s)

The second step is to configure the connection in the target Web Parts, the one(s) where results should be filtered by selected items of the source one. In my case the documents related to a specific product.
To achieve this, I add a new Web Part on the page and configure a base query to retrieve documents that will be filtered:

{{< image src="images/post/modern-search-focus-item-selection/refiner_mode_configuration.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

> You can connect multiple target Web Parts to a single source.

The connection with the source Search Results Web Part is done on the third configuration page in the property pane. From here you have to set:
1. The source search Results Web Part to use. If you select anything else than a search results Web Part, the configuration is simply reset.
2. The property of the selected item containing the value to use as filter. In my case, I select the managed property name corresponding to my column. 
3. The property of items used to match the selected value. In my case, because the two lists share the same column, I can simply use the same managed property name (`RefinableString04`).

  > To see the list of the available values, the source/target Web Part must have results retrieved and properties added in the _selected properties_ in the data source. <br>
  > <br>{{< image src="images/post/modern-search-focus-item-selection/property_list.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title"  webp="false" >}}
4. The selection mode. By default the filtering is done using **refinement filters** just like if you've selected a value from the Search Filters Web Part. However, this mode has two constraints:

    - The target Web Part must display results initially (because you can't refine empty results basically:/).
    - The target property (i.e. destination field name) must be a refinable managed property in the search schema. It won't work otherwise.
5. If you've enabled mutli selection in the source Web Part, the operator you want to use between selected values (AND/OR).

After doing these configurations, you should be able to filter (refine actually) values in the target Web Part.

{{< image src="images/post/modern-search-focus-item-selection/refiner_selectionmode_demo.gif" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

#### Use the manual selection mode

The default selection mode constraints you to display a set of results at page load in target Web Parts which is, in some cases, not the behavior you neccessarily want in terms of UI experience.
To bypass this limitation, a manual selection mode is available. When enabled, **it is up to you to configure the query using the query template field in the data source and available tokens**:

{{< image src="images/post/modern-search-focus-item-selection/manual_mode_toggle.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

{{< image src="images/post/modern-search-focus-item-selection/manual_mode_query.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

<ins>Query template in manual mode</ins>

`{? Path:{site.absoluteUrl}/Documentation {|RefinableString04:{filters.RefinableString04.valueAsText}} IsDocument:1}`

<ins>Explanation</ins>:

- `{? ... }`: Conditional operator. If a token is not resolved in the enclosed expression, the query won't be submitted to the search engine (meaning you won't get any default results).
- `{| ... }`: **'OR'** operator token that will expand the conditions with the OR KQL operator. Use `{& ... }` if you want and AND condition.
- `{filters.RefinableString04.valueAsText}`: Token that will be replaced dynamically by the selected item value. Only string values are supported here.

With this configuration, you won't get any result at page load but only when you will select an item in the source Web Part. To completely hide target Web Part(s), you can also enable the "" option.

{{< image src="images/post/modern-search-focus-item-selection/hide_option.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

### Connection with native SharePoint 'List' Or 'Library' Web Parts

The cool thing about the item selection feature is it supports connection with OOTB SharePoint Web Parts. It means you can configure a 'List' or 'Library' Web Part as source and use the PnP Search Results Web Part(s) as targets.

{{< image src="images/post/modern-search-focus-item-selection/ootb_webparts.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

> OOTB SharePoint Web Parts can only be used as source, not targets.

In my previous example, I could totally substitute the PnP Search Results displaying devices from he list by the list itself using hte 'List' Web Part. Also, because I've mapped the _'Related Product'_ managed metadata column to the property `RefinableString04` using the `ows_taxId_<columnName>`, I can perform the match based on the taxonomy term ID. Simple as that. The other configurations remaisn exactly the same, only the source changes.

{{< image src="images/post/modern-search-focus-item-selection/ootb_webpart_connection.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

This type of connection supports single and multi value selection as well:

{{< image src="images/post/modern-search-focus-item-selection/ootb_demo.gif" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

## Conclusion

The new item selection feature of PnP Modern Search solution now unlocks many new scenarios to build amazing dynamic search experiences. Looking forward to see how the community will use it!

For more information:
- [Item selection feature documentation](https://microsoft-search.github.io/pnp-modern-search/usage/search-results/connections/#how-to-configure-item-selection)
- [PnP Modern Search 4.5.4](https://github.com/microsoft-search/pnp-modern-search/releases/tag/4.5.4)