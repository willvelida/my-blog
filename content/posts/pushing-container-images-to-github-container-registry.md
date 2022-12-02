---
title: "Pushing container images to GitHub Container Registry"
date: 2022-12-01T20:47:30+13:00
draft: false
tags: ["GitHub","Containers","Docker","GitHub Actions"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dx2m0srq1t6q6g7iugx0.png
    alt: "GitHub and Docker logos"
    caption: 'You can push docker images to GitHub Container Registry using GitHub Actions'
---

In my job, I build a lot of samples that I share with customers to show them how things work. A lot of my customers are interested in Azure Container Apps, so I want to be able to provide them with samples with pre-built container images, without having to share the entire application source code as well (especially if I've got a bunch of basic microservices, that don't really need to be included in the sample).

Enter GitHub Container Registry! (GHCR) I've seen a couple of sample repos where container images were being referenced from GHCR, but I didn't know it worked or how to push images to it. Turns out, it was fairly straightforward.

In this post, I'll talk about what GHCR is, and how we can push container images to it using GitHub Actions!

## What is GitHub Container Registry?

GitHub Container Registry stores container images within your organization or personal account, and allows you to associate an image with a repository. It currently supports both the [Docker Image Manifest V2, Schema 2](https://docs.docker.com/registry/spec/manifest-v2-2/) and [Open Container Initiative (OCI) specifications](https://github.com/opencontainers/image-spec).

In GitHub, we can build and push our docker images to GHCR within a GitHub Actions workflow file and make those images available either privately or publicly (I'm making my images public for my samples).

Let's say I have the following Dockerfile:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
LABEL org.opencontainers.image.source="https://github.com/willvelida/dapr-store-app"
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["Store/Store.csproj", "Store/"]
RUN dotnet restore "Store/Store.csproj"
COPY . .
WORKDIR "/src/Store"
RUN dotnet build "Store.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "Store.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Store.dll"]
```

This Dockerfile is just building a simple Blazor Server application (It's a pretty generic Dockerfile for all ASP.NET Core applications). 

Instead of pushing this to Docker Hub or Azure Container Registry, we're going to set up a GitHub Actions workflow file to push this container image into GHCR.

Let's take a look at how we can authenticate to GHCR.

## Using the GITHUB_TOKEN to authenticate to GHCR

The recommended method for authenticating to GHCR is to use the ```GITHUB_TOKEN```. GitHub provides you with a token that you can use to authenticate on behalf of GitHub Actions. At the start of each workflow run, GitHub will automatically create a unique ```GITHUB_TOKEN``` secret to use in the workflow, which you can use to authenticate.

When GHCR was in Beta, you could use a Personal Access Token (PAT) to authenticate. You'd need to be careful about the permissions that you gave the PAT token. With ```GITHUB_TOKEN```, this comes with [sufficient permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token) needed to push container images to GHCR

## Using a Personal Access Token to authenticate to GHCR

I did have some trouble using the ```GITHUB_TOKEN``` initially, so to get started, I used a PAT token. To create one, go to **Settings/Developer settings**, click on *Personal access tokens/Tokens (classic)** and then click on **Generate new token**. To push images to GHCR, you only need the following permissions:

- **read:packages**
- **write:packages**
- **delete:packages**

Once you've created the PAT, you can store it as a repository secret inside your GitHub repository that contains your Dockerfile.

Within your GitHub Actions workflow file, you can then authenticate to GHCR using the following:

```yml
- name: 'Login to GitHub Container Registry'
        run: |
          echo $CR_PAT | docker login ghcr.io -u <Your-GitHub-username> --password-stdin
```

Since it's recommended to use ```GITHUB_TOKEN``` instead of PAT tokens, we'll use that moving forward.

## Creating a GitHub Actions workflow

Now that we've got a way to authenticate to GHCR, we can create a GitHub Actions workflow file to push our container image. Let's take a look at the following:

```yml
name: Deploy Images to GHCR

env:
  DOTNET_VERSION: '6.0.x'
  
on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
    build-store-project:
        runs-on: ubuntu-latest
        defaults:
          run:
            working-directory: './Store'
        steps:
          - name: 'Checkout GitHub Action'
            uses: actions/checkout@main
          - name: 'Setup dotnet'
            uses: actions/setup-dotnet@v1
            with:
              dotnet-version: ${{ env.DOTNET_VERSION }}
          - name: 'Install Dependencies'
            run: dotnet restore
          - name: 'Build project'
            run: dotnet build --no-restore
      push-store-image:
        runs-on: ubuntu-latest
        needs: [build-store-project]
        defaults:
          run:
            working-directory: './Store'
        steps:
          - name: 'Checkout GitHub Action'
            uses: actions/checkout@main
            
          - name: 'Login to GitHub Container Registry'
            uses: docker/login-action@v1
            with:
              registry: ghcr.io
              username: ${{github.actor}}
              password: ${{secrets.GITHUB_TOKEN}}
              
          - name: 'Build Inventory Image'
            run: |
              docker build . --tag ghcr.io/<your-GitHub-username>/store:latest
              docker push ghcr.io/<your-GitHub-username>/store:latest
```

The most important steps to highlight are authenticating to GHCR, and then pushing the container image.

To authenticate, we can use the ```docker/login-action```, target *ghrc.io* as the registry, and use our username (passed in as **${{ github.actor }}**) and our ```GITHUB_TOKEN``` as the password.

Once we've been authenticated, we can tag our image, using the format ```ghcr.io/<your-github-username>/<image-name>:<image-version>```.

## Making our image publicly accessible

Now by default, when we publish a package the visibility will be private. You can keep your images private if you want, but for my samples I want them to be publicly available.

To make them available to our repository, we need to add a ```LABEL``` command to our Dockerfile. You should do this underneath the first ```FROM``` command like so:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
LABEL org.opencontainers.image.source="https://github.com/<your-github-username>/<your-repo-name>"
```

This will make our images visible on our repository's main page, like so:

![GitHub repository with the packages section highlighted with a red rectangle](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/f7kgcolh0al8gdtqg9uq.png)

Click on the package that you want to make public, then go to **Package Settings**. In the **Danger Zone** (cue Kenny Loggins), click on **change visibility** and choose **Public** like so:

![GitHub Package settings for changing package visibility](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/429kchs4c4a50wj4c3h3.png)

Now that our package is public, we can pull it using *docker pull* like so:

```bash
docker pull ghcr.io/willvelida/store:latest
```

## Conclusion

In this post, we talked about what GHCR is, how we can authenticate and push images to it using to it using GitHub Actions and then those images public.

Hopefully this has helped you with publishing container images to GitHub Container Registry. For private container images, I'll still use Azure Container Registry, but authenticating and pushing images to GHCR for the purposes of samples looks like the way to go.

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️