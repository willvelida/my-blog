---
title: "Using GitHub Environment Files for Actions Workflow Outputs"
date: 2025-01-06
draft: false
tags: ["GitHub Actions", "DevOps", "GitHub"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ge59x2c1sq2dnmf0xsgz.png
    alt: "Using GitHub Environment Files for Actions Workflow Outputs"
    caption: "Sometime in the future, set-output commands in GitHub Actions will be depreciated! Here's how you can use Environment Files instead!"
---

For one of my side projects, I use GitHub Actions to build and deploy my various microservices to Azure. In my workflows, I'll often retrieve a value (such as getting the name of a container registry), and use the `set-output` command within my pipeline so that I can use the value of that output further down my pipeline.

This is an example of what using the `set-output` command would look like in your GitHub Action workflow:

```yaml
# Abbreviated file
- name: Get ACR Server
          id: getacrserver
          run: |
            loginServer=$(az acr list --resource-group ${{ secrets.resource-group-name }} --query "[0].loginServer" -o tsv)
            echo "::set-output name=loginServer::$loginServer"
  
        - name: Login to Azure Container Registry
          run: az acr login --name ${{ steps.getacrname.outputs.acrName }}

        - name: Build Docker image
          run: |
            docker build -t ${{ steps.getacrserver.outputs.loginServer }}/${{ inputs.app-name }}:${{ github.sha }} .
# Rest of workflow file
```

However, during the workflow execution, you may have seen some warnings pop up in the output that look like this:

```bash
The `set-output` command is deprecated and will be disabled soon. Please upgrade to using Environment Files. For more information see: https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/
```

I was using the `set-output` command a lot in my Actions workflows, so this warning [came up](https://github.com/willvelida/biotrackr/actions/runs/12608745908) a lot!

GitHub announced back in October 2022 that they would be [deprecating save-state and set-output commands](https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/) in GitHub Actions. A lot of folks are still using these commands, so they postponed this change in July 2023, but in this blog post, I'll talk about what GitHub Environment files are, and how you can use them in your GitHub Actions workflows to manage state and output.

## What are Environment Files in GitHub Actions?

During the execution of a GitHub Action workflow, the runner will generate temporary files that we can use to perform certain actions. The path to these files can be accessed and edited using GitHub's default environment variables.

The environment variables that we are interested in is the `GITHUB_OUTPUT` variable. This variable is the path to the temporary file on the GitHub runner that we can use to set the current step's output from workflow commands.

For more information on Environment Variables in GitHub Actions, [check the documentation](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/workflow-commands-for-github-actions#environment-files).

## Changing our workflow to use GITHUB_OUTPUT to set an output parameter

To set an output using `GITHUB_OUTPUT`, we can do that in our workflow using the following:

```bash
echo "{name}={value}" >> "$GITHUB_OUTPUT"
```

Applying this change to our workflow above, it'll now look like this:

```yaml
# Abbreviated file
- name: Get ACR name
          id: getacrname
          run: |
            acrName=$(az acr list --resource-group ${{ secrets.resource-group-name }} --query "[0].name" -o tsv)
            echo "acrName=$acrName" >> "$GITHUB_OUPUT"
  
        - name: Get ACR Server
          id: getacrserver
          run: |
            loginServer=$(az acr list --resource-group ${{ secrets.resource-group-name }} --query "[0].loginServer" -o tsv)
            echo "loginServer=$loginServer" >> "$GITHUB_OUTPUT"
# Rest of workflow file
```

Note that we still need an `id` in our `getacrname` step so that we can retrieve the output value later. All that changes here is that instead of using the `set-output` command, we have changed our workflow to use `$GITHUB_OUTPUT` instead.

Now that we've written our environment variable to `$GITHUB_OUTPUT`, we can use it in any subsequent step in a workflow job, like so:

```yaml
# Abbreviated file
- name: Get ACR name
          id: getacrname
          run: |
            acrName=$(az acr list --resource-group ${{ secrets.resource-group-name }} --query "[0].name" -o tsv)
            echo "acrName=$acrName" >> "$GITHUB_OUPUT"
  
        - name: Get ACR Server
          id: getacrserver
          run: |
            loginServer=$(az acr list --resource-group ${{ secrets.resource-group-name }} --query "[0].loginServer" -o tsv)
            echo "loginServer=$loginServer" >> "$GITHUB_OUTPUT"
  
        - name: Login to Azure Container Registry
          run: az acr login --name ${{ steps.getacrname.outputs.acrName }}
  
        - name: Build Docker image
          run: |
            docker build -t ${{ steps.getacrserver.outputs.loginServer }}/${{ inputs.app-name }}:${{ github.sha }} .
# Rest of workflow file
```

With this implemented, we observe that the warnings that we got when using `set-output` have [now disappeared](https://github.com/willvelida/biotrackr/actions/runs/12617070669).

## Wrapping up

As you can see, this is a fairly straightforward change. By using `$GITHUB_OUTPUT`, we can write variables that we need within our GitHub Actions workflow's subsequent steps, just like we would when using the `set-output` command.

There hasn't been a lot of activity from GitHub about **WHEN** they will depreciate `set-output`, but I like to be proactive about these things, so I recommend you be proactive too (just test the changes in your workflow before merging to main 😉)

If you have any questions about this, please feel free to reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)!

Until next time, Happy coding! 🤓🖥️