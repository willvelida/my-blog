---
title: "Understanding Microsoft Entra Agent ID"
date: 2026-02-27
draft: false
tags: ["Agents", "Security", "AI", "Identity", "Entra ID", "Entra Agent ID"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xszseoz0414quy25upuv.png
    alt: "As agents become more capable of making decisions, they require own identities to ensure proper governance, security, and accountability. Microsoft Entra Agent ID extends identity capabilities to agents, which help developers to build, govern, and protect agent identities."
    caption: "As agents become more capable of making decisions, they require own identities to ensure proper governance, security, and accountability. Microsoft Entra Agent ID extends identity capabilities to agents, which help developers to build, govern, and protect agent identities."
---

As agents become more capable of making decisions, they require own identities to ensure proper governance, security, and accountability. AI Agents are already being used in a variety of different workflows (perhaps you've deployed some yourself!). But as they become more capable and make autonomous decisions, there are several concerns and challenges that must be addressed:

- How do we authenticate and authorize agents?
- How do we govern them?
- Probably most important of all, how do we distinguish them from humans?!

Microsoft Entra Agent ID extends identity capabilities to agents, which help developers to build, govern, and protect agent identities. In this article, we're going to explore how Entra Agent ID gives agents security capabilities, such as control access policies, identity protection, and more.

## Core concepts

Let's cover the basic concepts of Microsoft Entra Agent ID, starting with **Agent identity**.

### Agent Identity

This is the primary account used by an AI agent to authenticate to various systems. It'll have an object ID and app ID which can be used for authentication and authorization. An agent identity is a special service principal in Entra ID.

Agent identities don't have a password or any other credential. They authenticate by presenting an access token issued to the service or platform that they run on.

Agent identities can be used to request agent tokens from Entra ID (the subject of the access token), receive incoming access tokens issued by Microsoft Entra ID (the audience of the access token being the agent identity), and request user tokens from Microsoft Entra ID for an authenticated user. The subject of the token is a user, while the actor is the agent identity.

An Agent Identity has a few key components

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/natfvm1c240vfhdre5h1.png)

Each agent identity has an `id` or object ID, which Entra generates the `id` and identifies the account within a Microsoft Entra tenant. As mentioned earlier, agent identities don't have passwords, but use other credentials to authenticate.

Agents have a display name which is surfaced in places like the Azure portal, Teams, Outlook etc.

Agent identities have sponsors, which represents the human user or group that's accountable for an agent. One reason for this is when security incidents happen, sponsors can be contacted to intervene.

All agent identities are created from reusable agent identity blueprints. This established the kind of agent and records metadata shared across all agent identities of a common kind.

Finally, agent identities can have an optional agent user. Some agents need access to systems that require a user account for authentication. These agent users are secondary accounts that Microsoft Entra tenants that have been decorated as an AI agent. It has a different `id` than the agent identity, but a 1:1 relationship is established between an agent identity and its agent user.

Let's dive a little deeper into these components now.

### Agent Identity Blueprints

**Agent identity blueprints** provide us with the template and management structure for creating and managing multiple agent identities. This serves as the parent of an agent identity. All agent identities in a Microsoft Entra ID are created from an agent identity blueprint.

Organizations can deploy many instances of an AI agent which pursues different goals and require different levels of access. However, these many instances will share certain characteristics, and blueprints record these common characteristics so that all agent identities created using the blueprint have a consistent configuration.

*Blueprints don't just hold information*, as they're also a special identity type in a Microsoft Entra ID tenant. Blueprints can provision or deprovision agent identities, and a blueprint will contain an *OAuth client ID* and *Credentials*, which are used to request access tokens from Entra ID, and a special Microsoft Graph permission that enables the blueprint to create agent identities in the tenant. This permission is `AgentIdentity.CreateAsManager`.

*Blueprints actually hold credentials for agent identities*, not the agent identity itself. When agents want to perform an operation, credentials that are configured on the blueprint are used to request an access token from Entra ID.

*Blueprints are logical containers for agent identities*, meaning that identity administrators can apply policies and settings to agent identity blueprints that take effect for all agent identities that are created using the blueprint.

Blueprints can contain a number of **Agent identity blueprint principals**, which is an object in Microsoft Entra Agent ID that represents the presence of an agent identity blueprint within a specific tenant. When we add a blueprint application to a tenant, Entra will create a principal object, which is the agent identity blueprint principal.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mdudfdm1s9o514k13zd0.png)

The principal has two important roles here. **Token Issuance**, which is when the agent identity blueprint is used to acquire tokens within a tenant and the resulting token's `oid` or object ID claim references the agent identity blueprint principal.

This ensures that any authentication or authorization performed by the agent identity blueprint is traceable to its principal object in the tenant.

**Audit Logging** is when actions performed by the agent identity blueprint are recorded in audit logs as being executed by the principal. This ensures that there is accountability and traceability for operations that were performed by the agent identity blueprint.

### Agent User

An agent user is a secondary account that an AI agent uses to authenticate to various systems. These are for when agents need to interact with systems that require user objects, essentially acting as a digital worker.

Agent users represent a subtype of user identity within Microsoft Entra. Agent users receive tokens with the claim `idtyp=user`, which allow them to access APIs and services that require user identities.

The agent user isn't created automatically. We have to explicitly create these and connect it to its parent identity. Each agent identity can have at most one associated agent user, and each agent user is linked to exactly one parent agent identity, and each parent identity is linked to exactly one agent identity blueprint app.

Agent users are also created using an agent identity blueprint. They are always associated to a specific agent identity upon creation. They have distinct unique identifiers, which is separate from the agent identity, and they can only authenticate by presenting a token issued to the associated agent identity.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d8dg69u20fsdd12kucij.png)

When we create agent users, we have to give the agent identity blueprint the permissions to do this, as agent users are optional. When the blueprint is granted proper permissions, blueprints can create an agent user and establish a parent relationship with a specific agent identity. The agent identity is the parent of the agent users.

Agent users can act as human-user accounts, but they have a slightly different authentication model. Authentication happens through credentials assigned to the agent user using **Federated identity credentials**. These are used for authenticating both the agent identity blueprint and agent identity itself.

Agent users are restricted to the credentials that they are provided through their parent relationships, rather than passwords. There are also impersonation mechanisms that agent identities have, as they can impersonate its child agent users.

### Agent registry

This is a centralized repository that maintains metadata about all registered agents within an organization. Using the registry, systems and services can discover agents based on their capabilities, roles, and other attributes.

Agent Registry integrates with Entra Agent ID and Core Directory to enforce identity and discovery policies, supports flexible mappings between agent cards and multiple agent instances, and is the single-source of truth for agent-related data.

The registry helps organizations secure agent discovery, apply Zero Trust principles, and maintain governance.

### Operation Patterns

The agent identity platform supports two primary patterns for how agents operate and authenticate:

1. Interactive agents sign-in as a user and take action in response to user prompts, usually via a chat interface. These agents will act on behalf of the signed-in user, utilizing that user's authorization to perform actions. Interactive agents are granted Entra delegated permissions that allow them to act on behalf of users. Tokens that are issued to interactive agents are often called user token.
2. Autonomous agents are agents that perform actions using their own identity, not a human one. These run in the background and take autonomous decisions about what actions to take. Tokens that are issued to autonomous agents are often called *agent tokens* when an agent identity is authenticated, or they can be called *agent user tokens* when an agent user is authenticated.

### Owners, sponsors, and managers

The agent identity platform introduces an administrative model that separates technical admin from business accountability. These roles include *owners*, *sponsors*, and *managers*.

Owners are essentially technical admins for agents, so they can handle operational and configuration aspects. Sponsors provide business accountability for the agents, and a manager is a human user, who acts as the hiring manager or operational owner for an agent user.

## Conclusion

Microsoft Entra Agent ID is still in preview, but it marks an important first step in ensuring that agents we deploy are secure and governable. I'll be exploring this area at greater depth in the near future, so I'll be creating more content as Entra Agent ID evolves!

If you have any questions about the content here, please feel free to reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️
