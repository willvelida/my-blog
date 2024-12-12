---
title: "Building a token refresh service for the Fitbit API with Container App Jobs"
date: 2024-12-12
draft: false
tags: ["Azure", "Azure Container Apps", "GitHub Actions", "DevOps", "IaC", "CSharp", "Containers", "Biotrackr"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u2grf7ekqzuaah3dzqgy.png
    alt: "Building a token refresh service for the Fitbit API with Container App Jobs"
    caption: "With Container App Jobs, we can containerized tasks for a finite duration for a variety of tasks, such as secret rotation"
---

A couple of years back, [I built a Fitbit API token refresh service using Azure Functions](https://dev.to/willvelida/building-a-token-refresh-service-for-the-fitbit-api-with-azure-functions-and-c-44mk). The purpose of that function was to refresh an access token that I can use to make calls to the Fitbit API and extract my own data using Azure Functions.

This was before Azure Container Apps even existed, so Azure Functions was really the only option. WebJobs was an alternative (after all, Functions are essentially built on top of Azure WebJobs), but that wasn't attractive since this was just an hobby project, and I didn't want to break the bank. The problem with Azure Functions is that you're restricted with the programming model. It's fine if you just need the plumbing done for you for simple integrations, but if you want the flexibility, it's not great.

[In May 2023](https://techcommunity.microsoft.com/blog/appsonazureblog/introducing-jobs-for-azure-container-apps/3826677?WT.mc_id=MVP_400037), Azure Container App Jobs went GA! Jobs are like Azure Function where they have a finite lifespan, and perform a single unit of work. We can run them manually, based on events, or on a schedule!

In this article, I'll talk about how I've built my Fitbit API token refresh service using Azure Container App Jobs, starting from implementing the application code to call the Fitbit API to generate an access token and storing the access token in Key Vault, to implementing setting up the infrastructure to deploy our service on Container App Jobs.

*If you just want to jump into the code and see how I built it, [check out the code my GitHub](https://github.com/willvelida/biotrackr)*

## What are Container App Jobs?

[Container App Jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs?tabs=azure-cli&WT.mc_id=MVP_400037) allow you to run containerized tasks that execute for a finite duration and exit. You can use jobs for a variety of tasks, such as Machine Learning, processing messages from a queue, or refreshing an access token.

Container App Jobs run in the same environment as normal Container Apps, which allow them to share the same networking and monitoring configuration. The Container App Environment is still the secure boundary around your Container App Jobs.

Container App Jobs define the default configuration that's used for each job execution. This includes things like the container image that the job uses, what resources it has (in terms of CPU and memory), and what to run. Job executions are a running instance of a job that's triggered either manually, on a schedule, or in response to events. A job will also have a certain amount of replicas that we can configure. In most cases, one replica is sufficient, but you can configure multiple replicas.

## Designing our token refresh service

Container App Jobs can run on a schedule, which is configured through CRON expressions. Using a CRON expression, we can kick off a job that makes a request to the Fitbit API to generate new refresh and access tokens, and then save those secrets into Key Vault.

Because the Fitbit API has a 8 hour lifetime for the access token, we need to be able to run this job every 6 hours (to give ourselves a couple of hours buffer just in case things go wrong).

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4o9sw8u8557qybvzp78f.png)

Once we've generated new refresh and access tokens, we can save a new version of our secrets in Key Vault that is used by our downstream applications making calls to various endpoints of the Fitbit API.

## How refresh tokens and access tokens work for the Fitbit API

Fitbit uses OAuth 2.0 to authorize users and authenticate to the Fitbit API. The OAuth 2.0 framework needs our apps that call the Fitbit API to obtain an Access Token. This is used for making HTTPS requests to the API.

Fitbit supports both Authorization Code Grant and Implicit Grant flows. For my project, I’m using the Authorization Code Grant flow. This is the recommended option for server-to-server communication using an application’s client secret.

Using the Authorization Code Grant Flow, a user will consent to share their Fitbit data with an application with some or all of the scopes. When this consent is given, Fitbit will give us an authorization code as part of a callback URL. Our app will exchange this code for an access and refresh tokens (the ones that we will store in Key Vault).

By default, these tokens will expire. The refresh token is used to obtain access to a new access and refresh token. Refresh tokens can only be used once, so we need to store the new access token and refresh token pair in Key Vault when these are regenerated.

For full details how to generate client credentials for the Fitbit API, please check out the following [documentation](https://dev.fitbit.com/build/reference/web-api/client-credentials/).

## Implementing our .NET Worker Service

Instead of an Azure Function, we can implement our refresh token as a [.NET Worker Service](https://learn.microsoft.com/dotnet/core/extensions/workers?WT.mc_id=MVP_400037). .NET worker services are perfect for creating services that work in the background, such as processing scheduled tasks.

To request new refresh and access tokens, we need to make to make calls to the Fitbit API (obviously....). In .NET work can use the [HTTPClient class](https://learn.microsoft.com/dotnet/api/system.net.http.httpclient?view=net-9.0&WT.mc_id=MVP_400037) to do so. Take a look at the following:

```csharp
using Azure.Security.KeyVault.Secrets;
using Biotrackr.Auth.Svc.Models;
using Biotrackr.Auth.Svc.Services.Interfaces;
using System.Net.Http.Headers;
using System.Text.Json;

namespace Biotrackr.Auth.Svc.Services
{
    public class RefreshTokenService : IRefreshTokenService
    {
        private readonly SecretClient _secretClient;
        private readonly HttpClient _httpClient;
        private readonly ILogger<RefreshTokenService> _logger;

        private const string RefreshTokenSecretName = "RefreshToken";
        private const string FitbitCredentialsSecretName = "FitbitCredentials";
        private const string FitbitTokenUrl = "https://api.fitbit.com/oauth2/token";

        public RefreshTokenService(SecretClient secretClient, HttpClient httpClient, ILogger<RefreshTokenService> logger)
        {
            _secretClient = secretClient;
            _httpClient = httpClient;
            _logger = logger;
        }

        public async Task<RefreshTokenResponse> RefreshTokens()
        {
            try
            {
                var fitbitRefreshToken = await GetSecretAsync(RefreshTokenSecretName);
                var fitbitClientCredentials = await GetSecretAsync(FitbitCredentialsSecretName);

                var tokens = await RequestNewTokensAsync(fitbitRefreshToken, fitbitClientCredentials);
                return tokens;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Exception thrown in {nameof(RefreshTokens)}");
                throw;
            }
        }

        public async Task SaveTokens(RefreshTokenResponse tokens)
        {
            try
            {
                _logger.LogInformation("Attempting to save tokens to secret store");
                await _secretClient.SetSecretAsync(RefreshTokenSecretName, tokens.RefreshToken);
                await _secretClient.SetSecretAsync("AccessToken", tokens.AccessToken);
                _logger.LogInformation("Tokens saved to secret store");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Exception thrown in {nameof(SaveTokens)}");
                throw;
            }
        }

        private async Task<string> GetSecretAsync(string secretName)
        {
            var secret = await _secretClient.GetSecretAsync(secretName);
            if (secret is null)
            {
                throw new NullReferenceException($"{secretName} not found in secret store");
            }
            return secret.Value.Value;
        }

        private async Task<RefreshTokenResponse> RequestNewTokensAsync(string refreshToken, string clientCredentials)
        {
            _httpClient.DefaultRequestHeaders.Clear();
            var uri = new UriBuilder(FitbitTokenUrl)
            {
                Query = $"grant_type=refresh_token&refresh_token={refreshToken}"
            };

            var request = new HttpRequestMessage(HttpMethod.Post, uri.Uri)
            {
                Content = new StringContent("")
            };
            request.Content.Headers.ContentType = new MediaTypeHeaderValue("application/x-www-form-urlencoded");
            request.Headers.Authorization = new AuthenticationHeaderValue("Basic", clientCredentials);

            var response = await _httpClient.SendAsync(request);
            response.EnsureSuccessStatusCode();
            _logger.LogInformation("Fitbit API called successfully. Parsing response");

            var content = await response.Content.ReadAsStringAsync();
            var tokens = JsonSerializer.Deserialize<RefreshTokenResponse>(content);

            return tokens;
        }
    }
}
```

In the `RequestNewTokensAsync` method, we define a POST method to the FitBit API that generates new refresh tokens for us by passing through our `refreshToken` and our `clientCredentials`. We use the `clientCredentials` to authenticate requests to the API, and once we make our POST request, we should receive a response from Fitbit with the tokens that we need to make authenticated calls.

This payload looks like the following:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9.eyJleHAiOjE0MzAzNDM3MzUsInNjb3BlcyI6Indwcm8gd2xvYyB3bnV0IHdzbGUgd3NldCB3aHIgd3dlaSB3YWN0IHdzb2MiLCJzdWIiOiJBQkNERUYiLCJhdWQiOiJJSktMTU4iLCJpc3MiOiJGaXRiaXQiLCJ0eXAiOiJhY2Nlc3NfdG9rZW4iLCJpYXQiOjE0MzAzNDAxMzV9.z0VHrIEzjsBnjiNMBey6wtu26yHTnSWz_qlqoEpUlpc",
  "expires_in": 3600,
  "refresh_token": "c643a63c072f0f05478e9d18b991db80ef6061e4f8e6c822d83fed53e5fafdd7",
  "token_type": "Bearer",
  "user_id": "26FWFL"
}
```

To parse this, I created a `RefreshTokenResponse` class to parse it correctly into a strong type.

```csharp
using System.Diagnostics.CodeAnalysis;
using System.Text.Json.Serialization;

namespace Biotrackr.Auth.Svc.Models
{
    [ExcludeFromCodeCoverage]
    public class RefreshTokenResponse
    {
        [JsonPropertyName("access_token")]
        public string AccessToken { get; set; }

        [JsonPropertyName("expires_in")]
        public int ExpiresIn { get; set; }

        [JsonPropertyName("refresh_token")]
        public string RefreshToken { get; set; }

        [JsonPropertyName("scope")]
        public string Scope { get; set; }

        [JsonPropertyName("token_type")]
        public string TokenType { get; set; }

        [JsonPropertyName("user_id")]
        public string UserType { get; set; }
    }
}
```

In the `RefreshTokens` method, we use the [Key Vault .NET SDK](https://learn.microsoft.com/dotnet/api/overview/azure/key-vault?view=azure-dotnet&WT.mc_id=MVP_400037) to retrieve our Fitbit Credentials and our Access Token. When we need to replace our new Access token, we use the Key Vault SDK to save the new version of our secret using this `SaveTokens` method.

With this Service created, we can then implement into our Background Service like so:

```csharp
using Biotrackr.Auth.Svc.Models;
using Biotrackr.Auth.Svc.Services.Interfaces;

namespace Biotrackr.Auth.Svc
{
    public class AuthWorker : BackgroundService
    {
        private readonly IRefreshTokenService _refreshTokenService;
        private readonly ILogger<AuthWorker> _logger;
        private readonly IHostApplicationLifetime _appLifetime;

        public AuthWorker(IRefreshTokenService refreshTokenService, ILogger<AuthWorker> logger, IHostApplicationLifetime appLifetime)
        {
            _refreshTokenService = refreshTokenService;
            _logger = logger;
            _appLifetime = appLifetime;
        }

        protected override async Task<int> ExecuteAsync(CancellationToken stoppingToken)
        {
            try
            {
                _logger.LogInformation($"Attempting to refresh FitBit Tokens: {DateTime.Now}");
                RefreshTokenResponse refreshTokenResponse = await _refreshTokenService.RefreshTokens();
                _logger.LogInformation($"FitBit Tokens refresh successful. Saving to Secret Store: {DateTime.Now}");
                await _refreshTokenService.SaveTokens(refreshTokenResponse);
                _logger.LogInformation($"FitBit Tokens saved successfully: {DateTime.Now}");
                return 0;
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown: {ex.Message}");
                return 1;
            }
            finally
            {
                _appLifetime.StopApplication();
            }
        }
    }
}

```

Now that we've implemented our code, let's turn our attention to the infrastructure.

## Setting up our Container App Job Infrastructure

To create our Container App job infrastructure, I decided to use Bicep. We can create a job using the following definition:

```bicep
resource authService 'Microsoft.App/jobs@2024-03-01' = {
  name: name
  location: location
  tags: tags
  properties: {
    environmentId: containerAppEnv.id
    configuration: {
      replicaTimeout: 600
      replicaRetryLimit: 3
      triggerType: 'Schedule'
      scheduleTriggerConfig: {
        cronExpression: '0 */6 * * *'
        parallelism: 1
        replicaCompletionCount: 1
      }
      registries: [
        {
          server: acr.properties.loginServer
          identity: uai.id
        }
      ]
    }
    template: {
      containers: [
        {
          name: name
          image: imageName
          resources: {
            cpu: json('0.25')
            memory: '0.5Gi'
          }
          env: [
            {
              name: 'keyvaulturl'
              value: keyVault.properties.vaultUri
            }
            {
              name: 'managedidentityclientid'
              value: uai.properties.clientId
            }
            {
              name: 'applicationinsightsconnectionstring'
              value: appInsights.properties.ConnectionString
            }
          ]
        }
      ]
    }
  }
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${uai.id}': {}
    }
  }
}
```

Container App Jobs live in the `Microsoft.App/jobs` namespace, and we can configure the following job settings:

- **Job Type** - which indicates the type of job that we're building. In our case, this is a `Schedule` job.
- **Replica Timeout** - This is the maximum time in seconds to wait for a replica to complete. I've set this for 10 minutes, which is more than enough for a simple API call.
- **Polling Interval** - The time in seconds to wait between polling for events.
- **Replica retry limit** - The maximum number of times to retry a failed replica. I've set this to 3 for my job.
- **Parallelism** - This setting configures the number of replicas to run per execution. For most jobs, you only need 1 replica.
- **Replica Completion count** - This is the number of replicas to complete successfully for the execution to succeed. This has to be equal or less than the parallelism value. Easy enough for me, it's set to 1.

Depending on the type of Container App job that you configure, you'll need to provide details on how it's going to be triggered. For scheduled CRON jobs, this is pretty straightforward.

To define a schedule, we pass through a CRON expression. This is just a string consisting of expressions that describe the details of a particular expression. The format of CRON expressions generally take the following:

```bash
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of the month (1 - 31)
│ │ │ ┌───────────── month (1 - 12 or JAN-DEC)
│ │ │ │ ┌───────────── day of the week (0 - 6 or SUN-SAT)
│ │ │ │ │
│ │ │ │ │
│ │ │ │ │
* * * * *
```

I use a tool like [crontab.guru](https://crontab.guru/) to help write a proper CRON expression.

Other than that, it's very similar to configuring a normal Container App. I'm using a user-assigned managed identity to authenticate to my Azure Container Registry, where I will pull the registry from. 

For my environment variables, I'm also using the user-assigned managed identity to authenticate to my Key Vault, so I'll pass through the endpoint of the Key Vault, as well as the Application Insights Connection String to send logs to.

The container image is passed through as a parameter, as I will be deploying both the application and infrastructure code together. As part of the pipeline, I'll build the container image, push it to ACR, and then pass the image name as a parameter to my Bicep template which will be deployed.

Here's a snippet of my GitHub Action workflow that's doing this:

```yaml
name: Deploy Auth Service

jobs:
  # Setup jobs

  build-container-image-dev:
      name: Build and Push Container Image
      needs: run-unit-tests
      uses: willvelida/biotrackr/.github/workflows/template-acr-push-image.yml@main
      with:
        working-directory: ./src/Biotrackr.Auth.Svc
        app-name: biotrackr-auth-svc
      secrets:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        resource-group-name: ${{ secrets.AZURE_RG_NAME_DEV }}

  retrieve-container-image-dev:
      name: Retrieve Container Image
      needs: build-container-image-dev
      runs-on: ubuntu-latest
      outputs:
        loginServer: ${{ steps.get-acr-server.outputs.loginServer }}
      steps:
        - name: Azure login
          uses: azure/login@v2
          with:
            client-id: ${{ secrets.AZURE_CLIENT_ID }}
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

        - name: Get ACR server
          id: get-acr-server
          run: |
            loginServer=$(az acr list --resource-group ${{ secrets.AZURE_RG_NAME_DEV }} --query "[0].loginServer" -o tsv)
            echo "::set-output name=loginServer::$loginServer"

  # Bicep setup jobs omitted

  deploy-dev:
        needs: [preview, retrieve-container-image-dev]
        name: Deploy Template to Dev
        uses: willvelida/biotrackr/.github/workflows/template-bicep-deploy.yml@main
        with:
          template-file: './infra/apps/auth-service/main.bicep'
          parameters-file: ./infra/apps/auth-service/main.dev.bicepparam imageName=${{ needs.retrieve-container-image-dev.outputs.loginServer }}/biotrackr-auth-svc:${{ github.sha }}
          scope: resourcegroup
          environment: dev
        secrets:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resource-group-name: ${{ secrets.AZURE_RG_NAME_DEV }}
```

To see the full GitHub Action workflow, check it out [here](https://github.com/willvelida/biotrackr/blob/main/.github/workflows/deploy-auth-service.yml).

## Viewing executions of our Container App job

Once our Container App job has been deployed, we can either wait for our job to execute on the schedule that we've defined, or invoke it manually.

We can view the **Execution History** of our job in the Azure Portal, which looks something like the following:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8bhsnvjnh19v2m1jntx1.png)

## Conclusion

I prefer Azure Container Apps over Azure Functions because it's just containers at the end of the day. I don't need to conform to a particular programming paradigm, and I have the flexibility to configure my application and connections the way I want to. (*Don't get me wrong, I'm not shitting on Azure Functions here. If you need a simple way to integrate between components in Azure, Functions is great for this*).

With Azure Container App jobs, I can create simple, containerized applications that run on schedules to perform background tasks. Bear in mind that this was just making a call to a Fitbit API to refresh some tokens. I've seen far too many people (you know who you are) using Azure Functions to perform long background running processes! It's not what it's designed for 🙅 

With Container App Jobs, you can run them on workload profiles to perform background tasks like Machine Learning jobs, without having to worry about [timeouts like you would in Azure Functions](https://learn.microsoft.com/azure/azure-functions/functions-scale?WT.mc_id=MVP_400037#timeout).

If you want to learn more about Azure Container App Jobs, check out the following resources:

- [Jobs in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/jobs?tabs=azure-cli&WT.mc_id=MVP_400037)
- [Create a job with Azure Container Apps](https://learn.microsoft.com/azure/container-apps/jobs-get-started-cli?pivots=container-apps-job-scheduled&WT.mc_id=MVP_400037)

If you want to see the code that I talked about, please check out my [GitHub](https://github.com/willvelida/biotrackr)

If you have any questions about this, please feel free to reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! I'm loving BlueSky at the moment. It has a much better UX and performance than Twitter these days.

Until next time, Happy coding! 🤓🖥️