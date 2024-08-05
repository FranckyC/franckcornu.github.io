---
title: "Hey Copilot, what about are my metadata?"
date: 2024-08-01T00:00:00.000Z
# post thumb
images:
 - images/post/copilot-studio-metadata/post_thumbnail.png
#author
author: "Franck Cornu"
# description
description: "Illustrates the limitation of Copilot to work with custom SharePoint metadata"
# Taxonomies
categories: ["copilot","sharepoint"]
tags: [post]
type: "featured" # available type (regular or featured)
draft: false
sidebar: right
index: false
showimage: false
_build:
  list: false
---

In this blog post, I wanted to share with you a use case I came across regarding my recent experience with Copilot Studio.

> **DISCLAIMER** </br></br> I'm not an "AI expert" by any means, just a developer who consumes AI services which is not quite the same :D.

## Context

I recently started a proof of concept to evaluate Copilot capabilities using Copilot Studio. The business goal was basically to help people working on an specific domain to navigate through SharePoint documentation content using a dedicated assistant answering questions about it.

All this information was stored as modern pages in multiple SharePoint sites using a custom information architecture. Several metadata were used for classification and search purposes (most of them being managed metadata defined in the taxonomy term store).

The main issue was, even with regular search experience with filters, users were still struggling to find the relevant information due to the amount of content, especially how things were related together through metadata. It was a perfect use case for a Copilot project.

## Costing and licences considerations for Copilot Studio

AI is cool but it is quite expensive and costs have to be justified and planned before implementing anything. As of today (August 2024), to build Copilots within Copilot Studio, you have two options:

- **Copilot for Microsoft 365**: this licence allows you to extend existing copilots by writing plugins, for instance making available specific data sources or actions.

    This plugin can be then published whithin the organisation for other users, requiring them to have a Copilot for Microsoft 365 licence as well. With this type of licence you can't build (actually share) a standalone Copilot (i.e a bot acessible by other users without Copilot licence). 

    For my use case, giving a full Copilot for Microsoft 365 licence for hundred of users was not an option. Despite this licence comes with unlimited messages quota and full integrations (like Word, Excel, etc.), most of the time, only few features are really useful. In my case I wanted an assistant accessible on-demand by anyone inside the company.

- **Copilot Studio licence(s)**: For Copilot Studio, you need two types of [licences](https://go.microsoft.com/fwlink/?linkid=2085130):

    - a _tenant_ licence coming with a "capacity pack" of 25 000 messages per month. This capacity is shared by all the standalone Copilots you create. According to Microsoft, a "billed message is a request or message sent to the copilot triggering an action and/or response". A Regular answer (Non-generative AI) costs 1 message but a generative AI (Gen AI) answer over your data costs 2 messages. It means you could create a Copilot without using any AI features. Talking about AI, the key element in Copilot Studio enabling the AI capability is called [generative answers](https://learn.microsoft.com/en-us/microsoft-copilot-studio/faqs-generative-answers). It is a special action available in the editor allowing you to use LLM over your data:
    
        > **Generative answer** </br></br>
        > It is not clear what model Microsoft use to do this, but it appears to be [GPT 3.5 Turbo](https://platform.openai.com/docs/models/gpt-3-5-turbo) as the documentation states corpus upon which the model was trained doesn't include data created after 2021. I don't see Microsoft using a more expensive GPT4 model here.
    
    - a _user_ licence (Free) allowing an user to author and publish Copilots in the Copilot Studio interface.

    The main interest of the Copilot Studio licence(s) is that users of your Copilots don't need to have a Copilot for Microsoft 365 licence to consume them. Also you can publish on multiple channels (Teams, web, etc.). Moreover, having a quota system reduces the risk to pay for unused features and allows you to be more in control regarding costs. When you see the usage growing, you can simply add more quotas by buying an other capacity pack, going steps by steps and measuring adoption along the way. It is also important to notice unused messages inside quotas do not carry over month to month. If you don't use your quotas, well, too bad for you. That's why monitoring is an important part of the process.

**IMPORTANT: It seems having a Copilot Studio tenant licence will allow users with Copilot for Microsoft 365 licences to author, publish and share a standalone Copilots in Teams only (other channels will be blocked)**. I didn't find any information about this but this is the behavior I noticed in my tenant.

## Building a copilot using Copilot Studio

Let's take a fictional example of a "Marvel" copilot, answering questions about Marvel's universe (characters, locations, etc.), the information being stored as modern pages in multiple SharePoint sites corresponding to each entity (i.e. character pages stored in a "Character" site, etc. ). 

> Note that I've been able to reproduce the same exact behavior with different type of content so it means you could reproduce it on your own tenant as well.

The experience of building a copilot with Copilot Studio is quite straightforward. I was able to select the SharePoint sites containing all the information to retrieve. I started with a very basic behavior with only one topic configured using the generative answer action and setting multiple SharePoint sites as knowledge sources (without using Copilot general knowledge). For the initial prompt, I started with this simple prompt (the Copilot must be configured to use Gen AI for conversations):

{{< image src="images/post/copilot-studio-metadata/copilot_parameters.png" caption="" alt="alter-text" position="center" class="img-fluid" title="Copilot settings"  webp="false" >}}

```
You are a personal assistant for company employees. 
You will answer questions about documentation from the SharePoint internal documentation.
Your answers will be precise and complete.
```
{{< image src="images/post/copilot-studio-metadata/copilot_prompt.png" caption="" alt="alter-text" position="center" class="img-fluid" title="Copilot prompt"  webp="false" >}}

### First test: questions about information present in the page content

I started with simple questions like asking for a custom specific information, like the full name of a character, an information present both in the page content and in a dedicated metadata (in this case a text field):

{{< image src="images/post/copilot-studio-metadata/page_metadata.png" caption="" alt="alter-text" position="center" class="img-fluid" title="Page metadata"  webp="false" >}}

As an example for this particular question: _"What is the full name of "Iron Man"?"_, I've tested two configurations: 

- **Configuration #1**: Targeting **all SharePoint sites** (i.e all entities like characters, locations, etc.) in a single generative answer: **no results**, Copilot saying there was not enough information in data sources to answer this question even though this information clearly appears both in page text and metadata.

- **Configuration #2**: Creating a topic dedicated to characters and targeting only the "Characters" SharePoint site. This time the Copilot gave the right answer (**but not all the time**).

The first lesson learned from that test is narrowing to specific sources seems to improve a lot the quality of responses. That is not really surprising. Behind the scenes, despite it is not officialy documented, Copilot only processes a subset of the results from the semantic index to generate its answer (probably around first 10 results for performances and tokens restrictions reasons). More sources you have, more results you'll get and potentially less relevant results to build the answer from anyway.

### Second test: questions about information present only in page metadata

In my second test, I wanted to evaluate how Copilot was able to leverage pages metadata. For this test, I focused my questions on characters having their full names only specified as metadata (i.e. as a SharePoint column) but not in the content of the page itself. As a result, the Copilot gave either **no answer** or completly **wrong answer** **EVERY. SINGLE. TIME.**. I rapidly figured out that my custom information architecture wasn't taken into account at all...

For a tool supposed to save you time, it was actually doing the opposite: the need to countercheck the information resulting to a lack of trust and therefore adoption :/.

#### Why custom metadata are not considered?

Knowing a little bit about Microsoft Search and Copilot, I think I figured out why my custom metadata were not taken into account.

**--SUPPOSITION MODE ON--**

Copilot is based on Microsoft Search and its [semantic index](https://learn.microsoft.com/en-us/microsoftsearch/semantic-index-for-copilot) to get data from various sources and generate answers from them. Behind the scenes, the semantic index generates [embeddings](https://huggingface.co/blog/getting-started-with-embeddings) from content (i.e. creating vectors from pieces of content). These embeddings are then compared together to find similarities according to a query and then returned to LLM to build an answer from these extracts. **As a Copilot customer, you have no control over this process nor the models used to generate embeddings from your content**.

On the Microsoft Search side now, when you define a data source, you must define its search schema and specify a "**content**" property. The content property represents the primary content of your item (ex: the content of a page or a document). SharePoint is no different from other sources here regarding search and also has a content property defined somewhere in its schema. In the case of a SharePoint modern page, the `CanvasContent1` property is likely used as content property to generate embeddings in the semantic index. This is where the raw content of the page is (i.e. text in the "Text" Web Part but also other components). 

Assuming this, then how Microsoft could possibly know what metadata from your custom schema are important and what are the ones who are not when generating embeddings? Short answer: they can't as their meaning is unique to you and your context so my guess is they are simply ignored in the process. In that sense, that is understandable because this could lead to incorrect information retrieved and generate noise for answers. Fair enough.

**--SUPPOSITION MODE OFF--**

#### Tricking the Copilot semantic index

Because I had no control over the semantic index and how embeddings are done, I tried multiple solutions to "trick" the semantic index and make the metadata information available to generate the answers. I tested few strategies:

1. **Metadata as text (original page)**

In this approach the goal was simply to explicity add the metadata as text in the original page at the very bottom: 

{{< image src="images/post/copilot-studio-metadata/ironman_metadata.png" caption="" alt="alter-text" position="center" class="img-fluid" title="Adding metadata to original page"  webp="false" >}}

Unlike the first test I did and even though the information was present in the page, the Copilot was not able to give the right answer. 

{{< image src="images/post/copilot-studio-metadata/ironman_metadata_questions.png" caption="" alt="alter-text" position="center" class="img-fluid" title="Copilot question - Test 1"  webp="false" >}}

Again, I think this is because of how embeddings are generated and how the content is structured in the page itself. It means even though the information is present in the page, **it does not guarantee it will be used for answers**. Well...
 
2. **Metadata in a new page.**

In that second approach, I completely created a new page using the same title as the original one and added the metadata as text in the content:

{{< image src="images/post/copilot-studio-metadata/ironman_metadata2.png" caption="" alt="alter-text" position="center" class="img-fluid" title="Adding metadata to original page"  webp="false" >}}

This time, asking the place or origin, the Copilot gave me the right answer!

{{< image src="images/post/copilot-studio-metadata/ironman_metadata2_questions.png" caption="" alt="alter-text" position="center" class="img-fluid" title="Copilot question - Test 2"  webp="false" >}}

It seems creating a new page using **the same title** but only including the metadata **as text** gives the correct output. With this (ugly) technique, I force, in a way, the semantic index to generate a separate embedding with this data and create a similarity with the original page through the "title" property (which must have a high importance in the process). This way this page comes high in the results and can be used for answers. Despite this approach seems to work, that is definitely not a long term satisfying option...

Here is a summary of strategies I tested and the Copilot outcomes:

| Approach      | Description                                                                       | Copilot outcomes           |
| --------------|-----------------------------------------------------------------------------------|----------------------------|
| #1 Metadata only | The information is only contained in the page metadata as SharePoint columns.       | Wrong or no answer at all.
| #2 Metadata as text (original page) | Metadata values are explicilty written in the page in a text Web Part at the bottom of the page. | Wrong or no answer at all.
| #3 Metadata as text (new page) | Metadata values are explicilty written in the page in a text Web Part in a new page with same title. | Correct answer.

{{< image src="images/post/copilot-studio-metadata/copilots_conversation_results.png" caption="" alt="alter-text" position="center" class="img-fluid" title="Conversation outcomes" webp="false" >}}

# Conclusion

For years, Microsoft told us to use metadata to classify the information to improve search and discoverability. Despite this is still a good practice for regular search experiences, with the new hype of GenAI, it seems these metadata are totally forgotten in Copilot experiences and custom information architectures are now becoming pretty much useless for such scenarios...

In my case, that was a big disapointement because, with Microsoft Search (Graph Connector in the context of Copilot if you prefer) many other sources (not only SharePoint) can contain custom schemas with a lot of valuable information that could be used to generate answers. As it is now, I can't use Copilot Studio for this type of requirements as the outcomes are not reliable.

**What about a custom Copilot...?**

Even with a custom Copilot scenario with Teams Toolkit and Teams AI library, there is no way to do RAG with the Microsoft Search semantic index programmaticaly as there is no (official) API.

**Why not using Azure Search instead...?**

Indexing SharePoint content into a other system like Azure Search with more flexibility could theorically work better but, **what would be the point to re-index content into an other system, at additional costs, while this data is already indexed into a service, Microsoft 365, I already pay for? An absolute nonsense**. Anyway [Azure Search SharePoint connector does not support .aspx pages](https://learn.microsoft.com/en-us/azure/search/search-howto-index-sharepoint-online#limitations-and-considerations) and do not even mention the complexity of managing content permissions.

**Ok but what about a custom Copilot plugin then..?**

Plugins are good to fetch live data based on a prompt and use returned data in the generated answer. Technically speaking, to get a specific metadata value to be considered in the answer, that could be a solution. However, it just requires few steps...:

1. Interpret the user query and detect every possible metadata he could ask for to trigger the plugin.
2. Identify uniquely the content that I need to get the metadata from based on that query, like the corresponding SharePoint pages. These information should theorically come from the semantic index in the first place.
3. In the plugin, call the Microsoft Graph API for all these pages based on the URL and get the metadata values.

What a complex implementation with an high chance of failure for something that could have been simply resolved by retrieving metadata value along the content in the first place...  

I really hope Microsoft will do something about it because the thousand of companies who invested in a custom SharePoint information architecture won't certainly appreciate this limitation. I already see some solutions that could help:

- In Copilot Studio, have a way to customize the underyling KQL Query and/or select the search managed properties to retrieve in the data source results and considered to generate answers.
- For custom scenarions, provide an API (like a grounding API) in Graph to consume the semantic index and get content extracts according to a KQL query. This API should also be able to select fields (i.e managed properties) to be retrieved so we could add them to the prompt doing our own RAG, for instance in a custom bot. This API could be tied to the Copilot Studio licences and quotas, to make sure you had to go through Copilot offer and not bypass the licensing restrictions.

# Bonus: other Copilot Studio limitations I found

Other than the metadata issues I faced, I also encoutered less impacful limitations in Copilot Studio like:

- You don't have access to the underlying query made to the search sematic index. You can't write your own KQL Query, for example to filter on specific well known search properties according to a topic to narrow retreived data.
- You can't reset a conversation using a Copilot in Teams. Because the Copilot bot keeps conversation history, it can be stuck in wrong context giving several wrong answers in a row. Sometimes, starting over could be nice when your Copilot starts to malfunction.
- Sometimes the language of copilot answers can change unexpectedly, even if the specified language is set to "English" in parameters and prompt instructions specify English only. For instance, asking a question about "[Bennet du Paris](https://marvel.fandom.com/wiki/Bennet_du_Paris_(Earth-616))" could return answers in French sometimes.
- You can't choose your LLM model. Currently it uses GPT 3.5 turbo (probably). To select your own, you can combine Copilot studio and Azure Open Azure AI but it requires additional costs and overlaps with Copilot Studio costs and value proposition. It is better to go with a custom copilot in this case.
- Not sure if this one is a bug but Copilot returns sometimes content from other sources than the ones specified in the action (ex: returning Confluence results even though this source hasn't been specified anywhere in the configuration).

Cheers!