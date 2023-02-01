---
title: "How to manage revisions in Azure Container Apps"
date: 2023-02-03
draft: true
tags: ["Azure","Azure Container Apps","Bicep", "Kubernetes", "csharp", "GitHub Actions"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/68qwdfvjgx7001m2vkjl.png
    alt: "Availability tests in Application Insights"
    caption: 'Automate your availability tests in App Service with Bicep'
---

Azure Container Apps implements versioning through revisions. Revisions are immutable snapshots of your container apps, which are automatically created whenever you make **revision-scope** changes to your container app.

You can use revisions for a variety of different uses, such as splitting traffic between your revisions to perform A/B testing, release new revisions in blue-green deployments, and being able to rollbacks to earlier versions of your app.

In this article, I'll describe what revisions are and how you can use them in different scenarios using a couple of demos

## What are revisions?

Container Apps implements versioning of your containers by creating revisions. When you first deploy your container app, the first revision is automatically created. When you make **revision-scope** changes, new revisions to your container app will be created.

### Revisions-scope vs application-scope changes

Revision-scope changes are changes that you make to the template of your container app. These include:

- The configuration of your container and the image that container uses.
- The scale rules that you apply to your containers.
- The suffix that you apply to your revision.

These changes only affect that particular revision. This is in constrast to **application-scope** changes. These affect all of your revisions, yet they don't create new revisions. When you change the configuration of your container app (not the container itself), you make an application-scope change. These include:

- The secrets that your container app uses.
- The revision mode your container app uses
- The ingress configuration of your container app.
- Any Dapr settings you apply to your container app.
- Credentials you use to authenticate to private container registries.

### Revision modes

In Container Apps, you can either deploy one revision at a time (single revision mode) or run multiple revisions of your container app simultaneously (multiple revision mode). 

In single revision mode, when you deploy a new version, the new version will automatically replace the active revision. In multiple revision mode, new versions of your container app are active alongside existing revisions of your container app. If your container app has external HTTP ingress enabled, you can control what percentage of traffic will be allocated to each active revision of your container app.

## How can you use revisions?

### A/B Testing

### Blue-green deployments

### Rollbacks

### Zero downtime deployments

## Conclusion

If you have any questions about this article, or have any feedback, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️