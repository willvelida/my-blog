---
title: "Enriching Indexers in Azure Cognitive Search with Skillsets"
date: 2023-09-27
draft: true
tags: ["Azure","Azure Cognitive Search","AI", "C#", "Bicep"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8r1p8foatub526q6lbpv.png
    alt: "Getting started with Azure Cognitive Search in C#"
    caption: "In this article, let's dive into Azure Cognitive Search and how we can query it using C# code!"
---

In Azure Cognitive Search, we use **indexers** to crawl searchable content from cloud data sources and populate our search indexes using mappings from our source data to our search index.

Indexers can also include **skillsets** that can contain one or more skills to call either built-in or external AI capabilities that are used to enrich documents that make their way to an index through its outputs.

In this article, I'll dive into what skillsets are and how they fit within the Indexer pipeline. Then, we'll jump into some code and see how we can transform unstructured text and images and create new documents that can be used in full-text search and knowledge mining scenarios.