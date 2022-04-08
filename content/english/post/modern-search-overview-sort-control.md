---
title: "New sort feature with the 4.6.0 PnP Modern Search version!"
date: 2022-04-08T00:00:00.000Z
# post thumb
images:
  - "images/post/modern-search-overview-sort-control/sort_control.png"
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

Since the `4.6.0` version of the PnP Modern Search solution, you can now configure sort fields for SharePoint/Microsoft Search data sources for both default sort and user sort (i.e sort results from the UI). As a result the previous _"Edit sort order"_ option for data sources has been renamed to _"Edit sort settings"_.

{{< image src="images/post/modern-search-overview-sort-control/sort_settings.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

That was a long awaited feature already present in v3 but not migrated to v4 due to the new architecture. This is now fixed :D.

## What changes compared to previous versions?

The first difference regarding the configuration is about the properties listed in the sort fields dropdown. Now the list is a static list representing all _"Sortable"_ properties in the SharePoint search schema:

{{< image src="images/post/modern-search-overview-sort-control/dropdown_list.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

However, if a property is missing or you need to use a custom property like `RefinableString` or `RefinableDate`, you can sill type the property name manually in the dropdown and press **'Enter'** to validate. Remember the property should be _"Sortable"_ in the search schema to get it work as there is no error validation anymore.

Then you can now decide for a property to be used either for default sorting or user sorting or both. For instance if you want to sort the results initialy by the `Created` managed property in ascending direction and also allow users to change this direction afterwards, you can set this property as user sort as well with a friendly name for users (i.e. the name that will appeart in the sort dropdown). You can also set other properties to only be sortable in the UI but not as default, like `RefinableString01`:

{{< image src="images/post/modern-search-overview-sort-control/sort_options.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

Once an user sort property is added in the configuration, a sort drop down control appears in the layout (if you don't set any _'User sort'_ property, the control does not appear):

{{< image src="images/post/modern-search-overview-sort-control/sort_control.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

> The dropdown control appears on all layouts (except the 'Details List')

To sort results on a property, simply click on the property you want to sort in the list **(1)** and then select the direction **(2)** ascending or descending.

{{< image src="images/post/modern-search-overview-sort-control/sort_control_usage.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

However, here are few things to know about the sort control:

- By default, when the page opens, the sort order will be the one represented by all properties set as _'Default sort'_ in the configuration (all combined together according to the definition order). 

- You can't preselect a property in the sort dropdown even if it is configured as a default sort property. It means users don't kow what is the default sort order. 

- Although you can set multiple properties as user sort in the configuration, you can sort **on only one property at a time in the UI**. 

- The default sort direction when you click on a property is always ascending (and you can't change this).

- You can reset the default sort order by clicking on the "default" option:

## Details list layout specificity

We've also updated the _Details List_ layout sort options to match this new behavior as it contains its own sort settings. Now in the column option you can make a specific column sortable by choosing among the properties you configured as _'User sort'_ in the data source sort configuration (and only these ones):

{{< image src="images/post/modern-search-overview-sort-control/details_list.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

In the update process, **we've also removed the static sorting behavior** since it was too confusing for users. Now when you sort a specific column, all resuts in the data set are sorted, not only the current page (i.e. a new search request is made):

{{< image src="images/post/modern-search-overview-sort-control/details_list_demo.gif" caption="" alt="alter-text" position="center" class="img-fluid" title="image title" webp="false" >}}

Hope you will find this new feature useful! Don't hesitate to provide your feedback in the [GitHub](https://github.com/microsoft-search/pnp-modern-search/discussions) repository.