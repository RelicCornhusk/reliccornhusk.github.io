---
layout: post
title: Creating Azure Pipelines from GitHub Repos using the API
date: 2024-08-03 21:11 -0400
categories: CI/CD ADO
tags: CI/CD ADO
---
For the past few months, I have been leading an enterprise-scale migration of a customer from Microsoft's legacy Team Foundation Server to GitHub/Azure DevOps. We are migrating several teams' source codes from TFVC to Git (using the git-tfs tool) and converting hundreds of XAML build definitions into Azure Pipelines YAML files. Suffice to say, we need to get creative at times to accomplish such a feat.

The ask from our client was to migrate all their source code to GitHub and to migrate their builds to Azure Pipelines, since the developers were already accustomed to using Azure DevOps. Given that they had hundreds of builds, I had to come up with a script that creates pipelines on Azure DevOps from the newly-converted YAML files stored on each team's private GitHub repository.

This should have been straightforward to do using the Azure DevOps REST API and some simple scripting but became a lot harder once I realized how incomplete Microsoft’s documentation was for that specific use-case. I couldn’t find a single GitHub issue or blog post of someone attempting this (hence this post!). After many tries and reverse engineering their API, I was successful in creating pipelines by doing a `POST https://dev.azure.com/{organization}/{project}/_apis/pipelines?api-version=7.2-preview.1` with the following body:

```json
{
    "name": "pipeline_name",
    "folder": "pipeline_folder",
    "configuration": {
        "type": "yaml",
        "path": "yaml_path",
        "repository": {
            "type": "github",
            "fullName": "github_repo",
            "defaultBranch": "github_branch",
            "url": "repo_url",
            "connection": {
                "id": "id_of_the_service_connection",
                "name": "name_of_the_service_connection",
                "url": "github_repo_url"
            }
        }
    }
}
```

Hopefully this helps someone trying to do the same thing!