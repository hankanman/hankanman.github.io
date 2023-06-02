---
title: AI Release Notes with GPT, DevOps & PowerShell - Part 1
description: I explore using PowerShell, Azure DevOps APIs, and OpenAI's GPT-3.5-turbo to automate the generation of release notes.
date: 2023-06-02T12:57:11.568Z
tags:
  - AI
  - Automation
  - Azure DevOps
  - GPT
  - OpenAI
  - PowerShell
  - DevOps
categories:
  - Automation
  - AI
slug: ai-release-notes-gpt-devops-powershell-part-1
pin: true
math: true
mermaid: true
img_path: /assets/posts/ai-release-notes-gpt-devops-powershell-part-1/
image:
  path: header.jpeg
keywords:
  - AI
  - DevOps
  - PowerShell
draft: true
---

## Introduction

Creating release notes is a critical task to keep stakeholders informed about the changes, fixes, and improvements in a new software version. This task, however, can be tedious, time-consuming, and prone to error if done manually. So I set out to build an AI release notes script leveraging PowerShell, Azure DevOps APIs, and OpenAI's GPT-3.5-Turbo LLM AI.

This blog post, the first in a multi-part series, will walk you through how we built the foundational parts of our script. It will cover the script setup, explaining each component and its role in generating detailed, pretty, yet succinct, release notes using an AI LLM to do the boring writing part for us.

## Prerequisites

To follow along with this blog post, you will need the following:

- An Azure DevOps Account & Project [Sign Up for Azure DevOps](https://azure.microsoft.com/services/devops/)
- A Personal Access Token (PAT) to access Azure DevOps APIs [How to Create a DevOps PAT](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows#create-a-pat)
- An OpenAI API Key [Sign Up for an API Key](https://platform.openai.com/account/api-keys)
- PowerShell 7 or later
- A PowerShell IDE (I use [Visual Studio Code](https://code.visualstudio.com/))
- A basic understanding of PowerShell
- A basic understanding of Azure DevOps
- A basic understanding of Markdown

## Getting Setup

The script I've developed is written in PowerShell and is designed to automate the generation of release notes. This process involves extracting and summarizing data from Azure DevOps, grouping them by work items, and creating a Markdown file with all this information.

### PowerShell Script Flow

To understand the script, let's look at the key components in order.

The script is broken down into several parts:

1. Creating the output file and establishing a document structure
2. Retrieving work items using queries from Azure DevOps
3. Sending the work items to GPT AI to summarise the work done
4. Sending the entire output to GPT AI to summarise the release notes

The full script is designed to be run as a scheduled task or as part of a DevOps pipeline. It can be run on-demand or on a schedule. The script will create a Markdown file with the release notes and a summary file with the summary of the release notes.

Before we get the script doing anything "AI Like", we need to set up the variables and create the output file.

### Creating Your Queries in Azure DevOps

How you set up your queries in Azure DevOps is up to you, but for the purposes of this article, we will need three queries for work items:

1. Released (In this release)
2. In Testing (Possibly Coming Next Release)
3. Known Issues (Head off those support calls)

This is just based on the project requirements and client expectations when I wrote it, but you can modify the script to suit your needs. You can use one query or 100 queries, it's up to you. For this part of the article we will just use the released query.

Rather than me explaining an ever-evolving platform like Azure DevOps, I will point you to the [Microsoft Documentation](https://learn.microsoft.com/en-us/azure/devops/boards/queries/using-queries?view=azure-devops&tabs=browser) on how to create queries. You might set up your queries based on tags, commits, pull requests, sprints or however else your team differentiates between releases.

### Establishing Variables

Firstly, the script begins by setting up necessary variables such as organization name, project name, solution name, and various API keys.

```powershell
# Initialize variables
# Replace the values below with your own
# The organization name is the name of your Azure DevOps
# organization this should be the same as the URL you use to
# access Azure DevOps https://dev.azure.com/<yourOrganizationName>
$orgName = "<yourOrganizationName>"
# The project name is the name of your Azure DevOps project
# this should be the same as the URL you use to access Azure
# DevOps https://dev.azure.com/<yourOrganizationName>/<yourProjectName>
$projectName = "<yourProjectName>"
# What is your product/solution/software called?
$solutionName = "<yourSolutionName>"
# What is the version of this release?
$releaseVersion = "1.0.0.0"
# The query IDs below are the IDs of the queries we created in
# DevOps, these GUIDs are appended to the URL of the query in DevOps
# The query that contains all the work items that were resolved
$releaseQuery = "00000000-0000-0000-0000-000000000000"
# We need a Personal Access Token (PAT) to access the Azure DevOps APIs
$pat = "<yourPAT>"
# We also need an OpenAI API Key to use the AI model
$openAIKey = "<yourOpenAIKey>"
```

We can of course turn these variables into parameters, but for the sake of simplicity, we will leave them as is for now, in later parts to the series we will explore how to make this script more dynamic and fit it into a DevOps pipeline.

Before we go any further and getting cool stuff to happen, we need to make sure that our variables are properly escaped. This is because we will be using these variables in URLs and we need to make sure that they are properly encoded to avoid any issues.

```powershell
# Encode $orgName and $projectName for URLs
$orgName = [System.Uri]::EscapeUriString($orgName)
$projectName = [System.Uri]::EscapeUriString($projectName)
# Encode $pat for Basic Authentication
$pat = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$($pat)"))
# Set the headers for the API calls
$headers = @{Authorization = ("Basic {0}" -f $pat) }
```

## Creating the Output File

Now that we have our variables set up, we can create the output file. This is the file that will contain the release notes. We will also create a summary file that will contain the summary of the release notes.

```powershell
# Define the path to the output file, the below will output to
# the a New Releases folder in the directory where the script is run
$outputFile = ".\Releases\$solutionName-v$releaseVersion.md"
# Create the output file using the path defined above
New-Item -ItemType file -Path $outputFile -Force | Out-Null
# Create the summary file for later use
$summaryFile = ".\Releases\summary.txt"
# Create the Summary file
New-Item -ItemType file -Path $summaryFile -Force | Out-Null
```

### Creating the Document Structure

Now that we have our output file, we can start writing stuff! We will start with the main header, version, and placeholders for the summary and table of contents.

```powershell
# Write the release notes header
"`n# Release Notes for $solutionName version v$releaseVersion`n`n## Summary`n`n<NOTESSUMMARY>`n`n## Quick Links`n`n<TABLEOFCONTENTS>" | Out-File -FilePath $outputFile -Encoding utf8 -Append
```

The "`n" is a line break, so we are writing the header, a line break, the summary header, a line break, and then a placeholder for the summary. We are then writing the table of contents header and a placeholder for the table of contents. Which gives us this:

![Release Notes Header](Release-Notes-Header.png)

**Note** our placeholders are hidden in the output

## Getting Our Work Items

Now that we have our output file, we can start getting our work items. We will start by getting the work items from our queries with the DevOps API.

```powershell
# Build the URI for the release query
$uri = "https://dev.azure.com/$orgName/$projectName/_apis/wit/wiql/$queryId`?api-version=6.0"
# Get the work items from the release query
$query = Invoke-RestMethod -Uri $uri -Headers $headers -Method Get
# Get the IDs of the work items from the query and join them into a comma separated string
$ids = $query.workItems.id -join ","

# Construct the URI to get the actual work items from the query with their expanded fields
$uri = "https://dev.azure.com/$orgName/$projectName/_apis/wit/workitems?ids=$ids&`$expand=all&api-version=6.0"
# Get the work items from the query
$workItems = Invoke-RestMethod -Uri $uri -Headers $headers -Method Get
# Write the number of work items found and the query URI to the console
Write-Output "Found $($workItems.count) work items."
# Add a nice header for our fixes
"`n### In This Release`n`n---`n" | Out-File -FilePath $outputFile -Encoding utf8 -Append
```

As a bonus you can output the work items to a json file so you can see what data is available like this:

```powershell
$workItems | ConvertTo-Json -Depth 10 | Out-File -FilePath '.\Releases\workItems.json' -Encoding utf8
```

This is useful because DevOps may have been customised with additional fields and you may want to include them in your release notes. You can find an example of this output in the assets for this article.

Here is a little snippet so you get the idea:

```json
{
  "count": 123,
  "value": [
    {
      "id": 1111,
      "rev": 1,
      "fields": {
        "System.Id": 1111,
        "System.AreaPath": "Example - Project Plus",
        "System.TeamProject": "Example - Project Plus",
        "System.RevisedDate": "9999-01-01T00:00:00Z",
        "System.IterationPath": "Example - Project Plus\\Example Sprint 1",
        "System.WorkItemType": "User Story",
        "System.State": "Done",
        "System.Reason": "Work finished",
        "System.Title": "As a User, I want a green button so I can click it",
        "Microsoft.VSTS.TCM.ReproSteps": "Navigate to the page and click the button",
        "Microsoft.VSTS.Common.AcceptanceCriteria": "The button should be green",
      }
    }
  ]
}
```

You will see a lot of fields that are prefixed with "System." and these are the default fields that come with DevOps. You will also see some fields that are prefixed with "Microsoft.VSTS." and these are the fields that are added when you create a new project in DevOps. You will also see some fields that are prefixed with "Custom." and these are the fields that are added when you customise your DevOps project.

We can use any of this data to create our release notes, but for now, we will stick to the default fields.

## Sending Our Work Items to our AI

Okay so now we have our work items from DevOps, we can start sending them to GPT to get our release notes. We will start by creating a for each loop that will send our each of work items to GPT and return the response. This is a bigger chunk of code, so read through the comments to see what is happening.

```powershell
# Loop through each work item
foreach ($workItem in $workItems.value) {
    # Print what the script is doing to the console
    # Note we now have access to all the properties returned by the API earlier
    Write-Output "Processing $($workItem.id) - $($workItem.fields.'System.Title')"

    # We need to give GPT a prompt to tell it what we want it to do
    # I have included the prompt i found to be most effective here, feel free to use it
    $prompt = "provide a single sentence of the work completed for the given devops work item details, ignore timestamps and links. Return only the description text with no titles, headers, or formatting, if there is nothing to describe, return 'Addressed', always assume that the task was completed. Do not list filenames or links"

    # We need to get the title, description, repro steps and acceptance criteria from the work item
    $content = "$($workItem.fields.'System.Title'), $($workItem.fields.'System.Description'), $($workItem.fields.'Microsoft.VSTS.TCM.ReproSteps'), $($workItem.fields.'Microsoft.VSTS.Common.AcceptanceCriteria')"

    # Now we need to construct the json payload for the OpenAI API
    $payload = @{
        "model"    = "gpt-3.5-turbo"
        "messages" = @(
            @{
                "role"    = "user"
                "content" = "$prompt`: $content"
            }
        )
    } | ConvertTo-Json

    # Now we can call the OpenAI API
    $summarizeResponse = Invoke-RestMethod -Method Post -Uri "https://api.openai.com/v1/chat/completions" -Body $payload -Header @{ "Content-Type" = "application/json"; "Authorization" = "Bearer $openAIKey" }

    # The API returns a list of choices, we want the first one
    $summary = " - $($summarizeResponse.choices[0].message.content)"

    # Get the work item ID
    $id = $workItem.id
    # Get the work item title
    $title = $workItem.fields.'System.Title'
    # Get the work item URL
    $url = $workItem.url

    # Write the work item ID and title to the output file
    "- [#$id]($url) **$title**$summary" | Out-File -FilePath $outputFile -Encoding utf8 -Append

    # Write the title and summary to the summary file
    $title + $summary | Out-File -FilePath $summaryFile -Encoding utf8 -Append
}
```

In this loop we are writing a prompt as a variable `$prompt` and storing the content of our DevOps work item to another variable `$workItem`. We then construct a json payload in `$payload` for the OpenAI API and send it off then store the response in `$summarizeResponse`. We then write the work item ID, title and summary to our output file by appending it to the end of the file. Finally we are taking a copy and writing to the `summary.txt` file.

After our for each loop let's give the user some feedback on how many work items were processed.

```powershell
# Update the user on progress
Write-Output "Finished processing $($workItems.count) work items."
```

Now we have some pretty comprehensive release notes, but we can do better. So now we will take everything we have done, and throw it at GPT again to give us a summary of the whole release.

```powershell
# Tell the user we are summarizing the release
Write-Output "Processing summary..."

# Get the final summary for header
# Retrieve all of our summary text
$summaryNotes = Get-Content -Path $summaryFile -Raw
# Call GPT to summarize the whole release

# Set a new prompt to use
$prompt = "review the following and write a summary of the work completed and key points for this release for the solution '$solutionName'"

# Now we need to construct the json payload for the AI with our new values
$payload = @{
    "model"    = "gpt-3.5-turbo"
    "messages" = @(
        @{
            "role"    = "user"
            "content" = "$prompt`: $summaryNotes"
        }
    )
} | ConvertTo-Json

# Now we can call the OpenAI API
$summarizeResponse = Invoke-RestMethod -Method Post -Uri "https://api.openai.com/v1/chat/completions" -Body $payload -Header @{ "Content-Type" = "application/json"; "Authorization" = "Bearer $openAIKey" }

# Get the first result for the summary from the response
$summary = $summarizeResponse.choices[0].message.content
```

We are doing the same thing as before, but this time we are using the summary file we created earlier. We are also using a different prompt for the AI to get a summary of the whole release. We then write the summary to the output file, as well as replacing our placeholders from earlier for the summary and table of contents.

```powershell
# Get the contents of the output file
$fileContents = Get-Content -Path $outputFile -Raw
# Replace the placeholders with the actual values
$fileContents = $fileContents.Replace("<NOTESSUMMARY>", $summary)
# Write the table of contents
$tableOfContents = "- [In This Release](#in-this-release)"
$fileContents = $fileContents.Replace("<TABLEOFCONTENTS>", $tableOfContents)

# Sometimes GPT will return a blank summary, if it does we need to replace it with a default
$fileContents = $fileContents.Replace(" - .", " - Addressed.")

# Write the file back to disk
$fileContents | Out-File -FilePath $outputFile -Encoding utf8 -Force

# Get the full path of the output file
$outputFilePath = (Get-Location).Path + $outputFile.Replace(".\","\")

# Convert the markdown to html and store a html verison of the notes as a bonus
(ConvertFrom-Markdown -Path $outputFilePath).Html | Out-File -FilePath $outputFilePath.Replace(".md",".html") -Encoding utf8 -Force

# Update the user on progress
Write-Output "Done!"
Write-Output "Markdown notes are here: $outputFilePath"
Write-Output "Html notes are here: $($outputFilePath.Replace(".md",".html"))"
```

We are replacing the placeholders we created earlier with the actual values, and then writing the file back to disk. We then convert the markdown file to html and write it to disk as well. Finally we give the user some feedback on where the files are located.

Your final markdown file should look something like this:

![Automating Release Notes Part 1 Final](Automating-Release-Notes-Part-1-Final.png)

We now have some great release notes from our AI assistant and we can run this against any project, query or work item type we want by changing the variables at the top of the script.

In the next part we will be making the script even better by adding some more features and making it more user friendly.

Stay tuned for Part 2!
