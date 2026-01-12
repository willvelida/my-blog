---
title: "How to Store Chat History Using External Storage in Microsoft Agent Framework"
date: 2026-01-12
draft: false
tags: ["Agents", "Microsoft Agent Framework", "AI", "CSharp", ".NET", "Azure Cosmos DB"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1ua9k4q86iehqaccpz4u.png
    alt: "For models that don't support underlying in-service memory capabilities, the Microsoft Agent Framework allows you to use third party solutions to store agent chat history."
    caption: "For models that don't support underlying in-service memory capabilities, the Microsoft Agent Framework allows you to use third party solutions to store agent chat history."
---

Chat history and memory allow agents to maintain context across conversations and remember user preferences, which enables agents to provide personalized experiences. Using the Microsoft Agent Framework, we can use in-memory chat message stores, persistent databases, and specialized memory services to cater to a variety of different use cases.

In this article, I'll show you a simple example of how we can use an Azure Cosmos DB Vector store to store conversations we have with an agent, and how we can retrieve conversations so that our agents can maintain context.

It's important to note that as I'm writing this (*12th January 2026*) the Microsoft Agent Framework is still in preview, so expect API's and features to change as time goes by.

## Chat History Options

There are two main supported scenarios:

1. **In-memory storage**: This is when an Agent is built on a service, like OpenAI Chat Completion, that doesn't support in-service storage of chat history. Agent Framework stores the full chat history in-memory using the `AgentThread` object. In order to store the chat history to a third-party store, we can create a custom `ChatMessageStore` implementation to do so.

2. **In-service storage**: This is when an Agent is built on a service, like Azure AI Foundry Persistent Agents, that requires in-service storage of chat history. The framework stores the ID of the remote chat history in the `AgentThread` object, and no other chat history storage options are supported.

This article covers implementing a custom `ChatMessageStore` so that we can use Azure Cosmos DB to persist our agent conversations.

The Agent Framework stores chat history in-memory using the `AgentThread` object. The full chat history is stored in the thread object which is provided to the service (for example, OpenAI Chat Completion), on every run as context.

OpenAI Chat Completion is an example of a service that doesn't support in-service storage of chat history, but what we can do is use the Agent Framework to replace the default in-memory storage of chat history with a third-party storage option, such as Cosmos DB.

## Configuring the Azure CosmosDB NoSQL Vector Store connector

To use Azure Cosmos DB to store our chat messages, we can use the Azure CosmosDB NoSQL Vector Store connector. We'll need to install the following NuGet package:

```bash
dotnet add package Microsoft.SemanticKernel.Connectors.CosmosNoSql --prerelease
```

Once you've installed the package, we can construct the instance to our Azure CosmosDB NoSQL Vector store directly like so:

```csharp
// Configure Cosmos DB as the Vector Store
var cosmosClient = new CosmosClient(config["cosmos-db-endpoint"], new AzureCliCredential(), new CosmosClientOptions()
{
    UseSystemTextJsonSerializerWithOptions = JsonSerializerOptions.Default
});
var database = cosmosClient.GetDatabase(config["database-name"]);
var vectorStore = new CosmosNoSqlVectorStore(database);
```

When initializing our `CosmosClient` manually, we have to specify the `UseSystemTextJsonSerializerWithOptions` due to limitations in the default serializer.

There are a couple of different methods you can use to create a connection to Azure Cosmos DB, which are described in the [documentation](https://learn.microsoft.com/en-us/semantic-kernel/concepts/vector-store-connectors/out-of-the-box-connectors/azure-cosmosdb-nosql-connector?pivots=programming-language-csharp#limitations).

## Creating our custom ChatMessage Store

To create a custom `ChatMessageStore`, we need to implement the abstract `ChatMessageStore` class and implement some required methods.

```csharp
internal class CosmosChatMessageStore : ChatMessageStore
{
    private readonly CosmosNoSqlVectorStore _cosmosVectorStore;

public CosmosChatMessageStore(
    CosmosNoSqlVectorStore cosmosVectorStore,
    JsonElement serializedStoreState,
    JsonSerializerOptions? jsonSerializerOptions = null)
    {
        this._cosmosVectorStore = cosmosVectorStore ?? throw new ArgumentNullException(nameof(cosmosVectorStore));
        if (serializedStoreState.ValueKind is JsonValueKind.String)
        {
            this.ThreadDbKey = serializedStoreState.Deserialize<string>();
            Console.WriteLine($"[CosmosChatMessageStore] Initialized with existing ThreadDbKey: {this.ThreadDbKey}");
        }
        else
        {
            Console.WriteLine($"[CosmosChatMessageStore] Initialized with no ThreadDbKey (new thread)");
        }
    }

    public string? ThreadDbKey { get; private set; }
}
```

The `ThreadDBKey` is a unique key that is used to identify the chat history in the vector store for subsequent calls. This key is persisted as part of the `AgentThread` state, allowing the thread to be resumed later and continued using the same chat history.

In the constructor, we pass through our `CosmosNoSqlVectorStore` instance that we'll use to store our conversation history. We also pass through the `JsonElement` for previously stored state that contains the `ThreadDBKey`. The purpose of this constructor is to either initiate a new conversation, should the `ThreadDBKey` not exist, or it will resume the conversation if it does.

We also need to implement three other methods:

- `InvokedAsync` - This is called at the end of the agent invocation to add new messages to the store.
- `InvokingAsync` - This returns messages in the correct chronological order.
- `Serialize` - The purpose of this method is to allow for the retrieval of strongly-typed services that might be provided by the `Microsoft.Agents.AI.ChatMessageStore`, including itself or any services it might be wrapping.

Let's take a look at the `InvokedAsync` method first:

```csharp
public override async ValueTask InvokedAsync(InvokedContext context, CancellationToken cancellationToken = default)
{
    this.ThreadDbKey ??= Guid.NewGuid().ToString("N");
    Console.WriteLine($"[InvokedAsync] Storing {context.ChatMessageStoreMessages.Count()} messages with ThreadDbKey: {this.ThreadDbKey}");
    
    var collection = this._cosmosVectorStore.GetCollection<string, ChatHistoryItem>("ChatHistory");
    await collection.EnsureCollectionExistsAsync(cancellationToken);
    
    var allNewMessages = context.RequestMessages.Concat(context.AIContextProviderMessages ?? []).Concat(context.ResponseMessages ?? []);

    await collection.UpsertAsync(allNewMessages.Select(x => new ChatHistoryItem()
    {
        Key = this.ThreadDbKey + x.MessageId,
        Timestamp = DateTimeOffset.UtcNow,
        ThreadId = this.ThreadDbKey,
        SerializedMessage = JsonSerializer.Serialize(x),
        MessageText = x.Text
    }), cancellationToken);
}
```

The purpose of this method is to save messages to the Vector Storage after we get a response from the agent. 

If this is the first message, it will generate a `ThreadDbKey`. It will then collect all messages from the current turn (`allNewMessages`), then it will create new records for each message before inserting them into our Cosmos DB store.

Now let's take a look at the `InvokingAsync` method:

```csharp
public override async ValueTask<IEnumerable<ChatMessage>> InvokingAsync(InvokingContext context, CancellationToken cancellationToken = default)
{
    Console.WriteLine($"[InvokingAsync] Retrieving messages for ThreadDbKey: {this.ThreadDbKey}");
    
    if (string.IsNullOrEmpty(this.ThreadDbKey))
    {
        Console.WriteLine($"[InvokingAsync] No ThreadDbKey - returning empty message list");
        return [];
    }

    var collection = this._cosmosVectorStore.GetCollection<string, ChatHistoryItem>("ChatHistory");
    await collection.EnsureCollectionExistsAsync(cancellationToken);
    
    var records = collection.GetAsync(
        x => x.ThreadId == this.ThreadDbKey, 10,
        new() { OrderBy = x => x.Descending(y => y.Timestamp) },
        cancellationToken);

    List<ChatMessage> messages = [];
    await foreach (var record in records)
    {
        Console.WriteLine($"[InvokingAsync] Retrieved record - Key: {record.Key}, Timestamp: {record.Timestamp}, MessageText: {record.MessageText?.Substring(0, Math.Min(50, record.MessageText?.Length ?? 0))}...");
        messages.Add(JsonSerializer.Deserialize<ChatMessage>(record.SerializedMessage!)!);
    }
    
    messages.Reverse();
    Console.WriteLine($"[InvokingAsync] Returning {messages.Count} messages in chronological order");
    
    return messages;
}
```

This reads messages and is called before we send requests to the AI Agent to provide our conversation history as context. It checks for the `ThreadDBKey`, queries Cosmos DB if it exists, and deserializes the messages from JSON back to `ChatMessage` objects

Finally, let's take a look at the `Serialize` method:

```csharp
public override JsonElement Serialize(JsonSerializerOptions? jsonSerializerOptions = null)
{
    Console.WriteLine($"[Serialize] Serializing ThreadDbKey: {this.ThreadDbKey}");
    return JsonSerializer.SerializeToElement(this.ThreadDbKey);
}
```

This method serializes the current object's state to a `JsonElement` using specified serialization options and it saves the `ThreadDBKey` so that the conversation can be resumed later.

Our entire `CosmosChatMessageStore` should look similar to this:

```csharp
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.VectorData;
using Microsoft.SemanticKernel.Connectors.CosmosNoSql;
using System.Text.Json;

namespace CosmosDBChatHistory
{
    internal class CosmosChatMessageStore : ChatMessageStore
    {
        private readonly CosmosNoSqlVectorStore _cosmosVectorStore;

        public CosmosChatMessageStore(
            CosmosNoSqlVectorStore cosmosVectorStore,
            JsonElement serializedStoreState,
            JsonSerializerOptions? jsonSerializerOptions = null)
        {
            this._cosmosVectorStore = cosmosVectorStore ?? throw new ArgumentNullException(nameof(cosmosVectorStore));
            if (serializedStoreState.ValueKind is JsonValueKind.String)
            {
                this.ThreadDbKey = serializedStoreState.Deserialize<string>();
                Console.WriteLine($"[CosmosChatMessageStore] Initialized with existing ThreadDbKey: {this.ThreadDbKey}");
            }
            else
            {
                Console.WriteLine($"[CosmosChatMessageStore] Initialized with no ThreadDbKey (new thread)");
            }
        }

        public string? ThreadDbKey { get; private set; }

        // AddMessageAsync
        public override async ValueTask InvokedAsync(InvokedContext context, CancellationToken cancellationToken = default)
        {
            this.ThreadDbKey ??= Guid.NewGuid().ToString("N");
            Console.WriteLine($"[InvokedAsync] Storing {context.ChatMessageStoreMessages.Count()} messages with ThreadDbKey: {this.ThreadDbKey}");
            
            var collection = this._cosmosVectorStore.GetCollection<string, ChatHistoryItem>("ChatHistory");
            await collection.EnsureCollectionExistsAsync(cancellationToken);
            
            var allNewMessages = context.RequestMessages.Concat(context.AIContextProviderMessages ?? []).Concat(context.ResponseMessages ?? []);

            await collection.UpsertAsync(allNewMessages.Select(x => new ChatHistoryItem()
            {
                Key = this.ThreadDbKey + x.MessageId,
                Timestamp = DateTimeOffset.UtcNow,
                ThreadId = this.ThreadDbKey,
                SerializedMessage = JsonSerializer.Serialize(x),
                MessageText = x.Text
            }), cancellationToken);
        }

        // GetMessagesAsync
        public override async ValueTask<IEnumerable<ChatMessage>> InvokingAsync(InvokingContext context, CancellationToken cancellationToken = default)
        {
            Console.WriteLine($"[InvokingAsync] Retrieving messages for ThreadDbKey: {this.ThreadDbKey}");
            
            if (string.IsNullOrEmpty(this.ThreadDbKey))
            {
                Console.WriteLine($"[InvokingAsync] No ThreadDbKey - returning empty message list");
                return [];
            }

            var collection = this._cosmosVectorStore.GetCollection<string, ChatHistoryItem>("ChatHistory");
            await collection.EnsureCollectionExistsAsync(cancellationToken);
            
            var records = collection.GetAsync(
                x => x.ThreadId == this.ThreadDbKey, 10,
                new() { OrderBy = x => x.Descending(y => y.Timestamp) },
                cancellationToken);

            List<ChatMessage> messages = [];
            await foreach (var record in records)
            {
                Console.WriteLine($"[InvokingAsync] Retrieved record - Key: {record.Key}, Timestamp: {record.Timestamp}, MessageText: {record.MessageText?.Substring(0, Math.Min(50, record.MessageText?.Length ?? 0))}...");
                messages.Add(JsonSerializer.Deserialize<ChatMessage>(record.SerializedMessage!)!);
            }
            
            messages.Reverse();
            Console.WriteLine($"[InvokingAsync] Returning {messages.Count} messages in chronological order");
            
            return messages;
        }

        public override JsonElement Serialize(JsonSerializerOptions? jsonSerializerOptions = null)
        {
            Console.WriteLine($"[Serialize] Serializing ThreadDbKey: {this.ThreadDbKey}");
            return JsonSerializer.SerializeToElement(this.ThreadDbKey);
        }
    }
}

sealed class ChatHistoryItem
{
    [VectorStoreKey]
    public string? Key { get; set; }
    [VectorStoreData]
    public string? ThreadId { get; set; }
    [VectorStoreData]
    public DateTimeOffset? Timestamp { get; set; }
    [VectorStoreData]
    public string? SerializedMessage { get; set; }
    [VectorStoreData]
    public string? MessageText { get; set; }
}
```

## Using our custom ChatMessageStore in our agent

To use the custom `ChatMessageStore`, we'll need to provide this using the `ChatMessageStoreFactory` when creating the agent like so:

```csharp
AIAgent fitnessAgent = new AzureOpenAIClient(
    new Uri(config["foundry-endpoint"]),
    new AzureCliCredential())
    .GetChatClient("gpt-4.1-mini")
    .CreateAIAgent(new ChatClientAgentOptions
    {
        Name = "Fitness Agent",
        ChatOptions = new ChatOptions()
        {
            Instructions = "You are a helpful fitness assistant."
        },
        ChatMessageStoreFactory = ctx =>
        {
            // This is our custom `ChatMessageStore`
            return new CosmosChatMessageStore(vectorStore, ctx.SerializedState, ctx.JsonSerializerOptions);
        }
    });
```

With our `ChatMessageStore` provided to our agent, we can interact with our agent and create a new thread that will hold conversation state that will be persisted to Cosmos DB.

Within our `CosmosChatMessageStore`, we can use the `ThreadDbKey` to restore our conversation state and reload it into our resumed thread by deserializing the state into a `JsonElement`.

```csharp
// First conversation
Console.WriteLine("=== First Conversation ===\n");
AgentThread agentThread = fitnessAgent.GetNewThread();
Console.WriteLine(await fitnessAgent.RunAsync("Give me 5 exercises for my legs", agentThread));

// Serialize and persist the thread state
JsonElement serializedThread = agentThread.Serialize();
string threadStateJson = JsonSerializer.Serialize(serializedThread, new JsonSerializerOptions() { WriteIndented = true });

Console.WriteLine("\n--- Serialized Thread State ---");
Console.WriteLine(threadStateJson);
Console.WriteLine("--- End Serialized Thread State ---\n");

// Resume the conversation
Console.WriteLine("\n=== Resuming Conversation ===\n");
JsonElement restoredThreadState = JsonSerializer.Deserialize<JsonElement>(threadStateJson);
AgentThread resumedThread = fitnessAgent.DeserializeThread(restoredThreadState);

Console.WriteLine(await fitnessAgent.RunAsync("Using the same exercises, give me a workout plan that I can use as part of a weight loss program", resumedThread));

var messageStore = resumedThread.GetService<CosmosChatMessageStore>();
Console.WriteLine($"\nThread is stored in vector store under key: {messageStore.ThreadDbKey}");
```

## Running our agent

Now that we've configured our agent to use our custom `ChatMessageStore`, let's see it in action. I've created a fitness agent that I'll ask to suggest 5 leg exercises, and then create a weight loss program using those same exercises. We should the agent respond, and our conversation state being persisted to Cosmos DB.

To run the agent, we can use `dotnet run`.

After our first request, we should see the following response:

```text
=== First Conversation ===

[CosmosChatMessageStore] Initialized with no ThreadDbKey (new thread)
[InvokingAsync] Retrieving messages for ThreadDbKey:
[InvokingAsync] No ThreadDbKey - returning empty message list
[InvokedAsync] Storing 0 messages with ThreadDbKey: 3d5a0472eec6458899b0caa1e17786c7
Sure! Here are 5 effective leg exercises you can try:

1. **Squats**
   - Targets: Quadriceps, hamstrings, glutes
   - How to: Stand with feet shoulder-width apart, lower your hips down and back as if sitting in a chair, keep chest up, then stand back up.

2. **Lunges**
   - Targets: Quadriceps, hamstrings, glutes, calves
   - How to: Step forward with one leg, lower your body until both knees are bent at 90 degrees, push back to starting position, then switch legs.

3. **Deadlifts**
   - Targets: Hamstrings, glutes, lower back
   - How to: With feet hip-width apart, hold weights (optional), hinge at the hips while keeping back flat, lower the weights down, then stand back up.

4. **Calf Raises**
   - Targets: Calves
   - How to: Stand on the edge of a step or flat ground, raise your heels as high as possible, then lower back down slowly.

5. **Step-Ups**
   - Targets: Quadriceps, hamstrings, glutes
   - How to: Step onto a bench or sturdy platform with one foot, press through the heel to lift your body up, then step down and switch legs.

Let me know if you'd like tips on reps and sets!
[Serialize] Serializing ThreadDbKey: 3d5a0472eec6458899b0caa1e17786c7

--- Serialized Thread State ---
{
  "storeState": "3d5a0472eec6458899b0caa1e17786c7"
}
--- End Serialized Thread State ---
```

A `ThreadDbKey has been created, which the agent will use to retrieve our messages as part of its context when answering our second question. We should see the following response:

```text
=== Resuming Conversation ===

[CosmosChatMessageStore] Initialized with existing ThreadDbKey: 3d5a0472eec6458899b0caa1e17786c7
[InvokingAsync] Retrieving messages for ThreadDbKey: 3d5a0472eec6458899b0caa1e17786c7
[InvokingAsync] Retrieved record - Key: 3d5a0472eec6458899b0caa1e17786c7chatcmpl-Cx242P0pnZcK7Ucoy6FrX4CV6o9cB, Timestamp: 12/01/2026 2:35:27 AM +00:00, MessageText: Sure! Here are 5 effective leg exercises you can t...
[InvokingAsync] Retrieved record - Key: 3d5a0472eec6458899b0caa1e17786c7, Timestamp: 12/01/2026 2:35:27 AM +00:00, MessageText: Give me 5 exercises for my legs...
[InvokingAsync] Returning 2 messages in chronological order
[InvokedAsync] Storing 2 messages with ThreadDbKey: 3d5a0472eec6458899b0caa1e17786c7
Certainly! Here's a beginner-friendly leg workout plan using those exercises, designed to support your weight loss goals by combining strength and calorie-burning movements. Aim to do this workout 2-3 times per week, alongside a balanced diet and regular cardio.

### Leg Workout for Weight Loss

**Warm-up (5-10 minutes):**
- Light jogging or brisk walking
- Leg swings and bodyweight squats

---

**Workout:**

1. **Squats**
   - 3 sets of 12-15 reps
   - Rest 30-45 seconds between sets

2. **Lunges** (alternating legs)
   - 3 sets of 10-12 reps per leg
   - Rest 30-45 seconds between sets

3. **Deadlifts** (use light to moderate weights or just bodyweight at first)
   - 3 sets of 10-12 reps
   - Rest 45 seconds between sets

4. **Calf Raises**
   - 3 sets of 15-20 reps
   - Rest 30 seconds between sets

5. **Step-Ups** (alternating legs)
   - 3 sets of 10-12 reps per leg
   - Rest 30-45 seconds between sets

---

### Additional Tips:
- Move quickly between exercises to keep your heart rate up, turning this into a circuit if you can.
- After completing all 5 exercises, rest 1-2 minutes, then repeat the circuit 2-3 times based on your fitness level.
- Include 20-30 minutes of moderate cardio (like walking, cycling, or swimming) on your non-strength days.

Would you like me to help create a full weekly workout schedule or include nutrition tips?

Thread is stored in vector store under key: 3d5a0472eec6458899b0caa1e17786c7
```

As we can see, the conversation has been reloaded using our `ThreadDBKey` (3d5a0472eec6458899b0caa1e17786c7) and the agent has created an exercise plan based of the exercises it suggested in it's first answer.

In our Cosmos DB collection, our conversation history will be persisted as documents that look like our `ChatHistoryItem` object. Here's one document as an example:

```json
{
    "ThreadId": "3d5a0472eec6458899b0caa1e17786c7",
    "Timestamp": "2026-01-12T02:35:27.7616746+00:00",
    "SerializedMessage": "{\"AuthorName\":\"Fitness Agent\",\"CreatedAt\":\"2026-01-12T02:35:22+00:00\",\"Role\":\"assistant\",\"Contents\":[{\"$type\":\"text\",\"Text\":\"Sure! Here are 5 effective leg exercises you can try:\\n\\n1. **Squats**  \\n   - Targets: Quadriceps, hamstrings, glutes  \\n   - How to: Stand with feet shoulder-width apart, lower your hips down and back as if sitting in a chair, keep chest up, then stand back up.\\n\\n2. **Lunges**  \\n   - Targets: Quadriceps, hamstrings, glutes, calves  \\n   - How to: Step forward with one leg, lower your body until both knees are bent at 90 degrees, push back to starting position, then switch legs.\\n\\n3. **Deadlifts**  \\n   - Targets: Hamstrings, glutes, lower back  \\n   - How to: With feet hip-width apart, hold weights (optional), hinge at the hips while keeping back flat, lower the weights down, then stand back up.\\n\\n4. **Calf Raises**  \\n   - Targets: Calves  \\n   - How to: Stand on the edge of a step or flat ground, raise your heels as high as possible, then lower back down slowly.\\n\\n5. **Step-Ups**  \\n   - Targets: Quadriceps, hamstrings, glutes  \\n   - How to: Step onto a bench or sturdy platform with one foot, press through the heel to lift your body up, then step down and switch legs.\\n\\nLet me know if you\\u0027d like tips on reps and sets!\",\"Annotations\":null,\"AdditionalProperties\":null}],\"MessageId\":\"chatcmpl-Cx242P0pnZcK7Ucoy6FrX4CV6o9cB\",\"AdditionalProperties\":null}",
    "MessageText": "Sure! Here are 5 effective leg exercises you can try:\n\n1. **Squats**  \n   - Targets: Quadriceps, hamstrings, glutes  \n   - How to: Stand with feet shoulder-width apart, lower your hips down and back as if sitting in a chair, keep chest up, then stand back up.\n\n2. **Lunges**  \n   - Targets: Quadriceps, hamstrings, glutes, calves  \n   - How to: Step forward with one leg, lower your body until both knees are bent at 90 degrees, push back to starting position, then switch legs.\n\n3. **Deadlifts**  \n   - Targets: Hamstrings, glutes, lower back  \n   - How to: With feet hip-width apart, hold weights (optional), hinge at the hips while keeping back flat, lower the weights down, then stand back up.\n\n4. **Calf Raises**  \n   - Targets: Calves  \n   - How to: Stand on the edge of a step or flat ground, raise your heels as high as possible, then lower back down slowly.\n\n5. **Step-Ups**  \n   - Targets: Quadriceps, hamstrings, glutes  \n   - How to: Step onto a bench or sturdy platform with one foot, press through the heel to lift your body up, then step down and switch legs.\n\nLet me know if you'd like tips on reps and sets!",
    "id": "3d5a0472eec6458899b0caa1e17786c7chatcmpl-Cx242P0pnZcK7Ucoy6FrX4CV6o9cB",
    "_rid": "s5xYAJun3qoCAAAAAAAAAA==",
    "_self": "dbs/s5xYAA==/colls/s5xYAJun3qo=/docs/s5xYAJun3qoCAAAAAAAAAA==/",
    "_etag": "\"6b0025f9-0000-1a00-0000-69645def0000\"",
    "_attachments": "attachments/",
    "_ts": 1768185327
}
```

## Conclusion

In this article, we talked about how we can create a `CustomMessageStore` so that we can store agent chat history in external storage. 

While I used Azure Cosmos DB in this article, there are a number of other connectors that you can use to store conversation history for agents like SQL Server, Redis, MongoDB and more! Take a look at the [docs](https://learn.microsoft.com/en-us/semantic-kernel/concepts/vector-store-connectors/out-of-the-box-connectors/?pivots=programming-language-csharp) for more information.

If you have any questions about the content here, please feel free to reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️