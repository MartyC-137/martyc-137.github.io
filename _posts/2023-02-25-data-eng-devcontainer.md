---
title: "Building a devcontainer for data engineering"
date: 2023-02-23T13:18:30-04:00
categories:
  - blog
tags:
  - Docker
  - Python
  - SQL
  - dbt
---

Recently I discovered [`devcontainers`](https://code.visualstudio.com/docs/devcontainers/containers), which allow to to configure a docker container as a complete development environment using either [Visual Studio Code](https://code.visualstudio.com/) or [GitHub Codespaces](https://github.com/features/codespaces). We all know how difficult it can be to configure an environment, install all of the required dependencies for a project, etc, so I thought it would be great to set up a reproducible environment for my team to use at work. We use Snowflake, Python (and various libraries such as [`Snowpark`](https://docs.snowflake.com/en/developer-guide/snowpark/python/setup)), SQL Server, dbt, etc. This will definitely be a moving target as we start to use new tools/deprecate others, but it is easy to add/remove features to the container after your initial setup.

If you're a data engineer, this guide will help you set up a development environment, and will hopefully give you the skills required to add anything you might need to your own `devcontainer`!

## Getting Started
To continue, make sure you have [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed OR use [GitHub Codespaces](https://github.com/features/codespaces).

**Option 1: Local VS Code**

1. Clone the [repo](https://github.com/MartyC-137/Data-Engineering-Devcontainer) and connect to it in VS Code:

```bash
$ cd your/desired/repo/location
$ git clone https://github.com/MartyC-137/DataEng_devcontainer
```

1. Download the [`Dev Containers`](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension from the VS Code marketplace

2. Press Cmd + Shift + P (Mac) or Ctrl + Shift + P (Windows) to open the Command Pallette. Type in `Dev Containers: Open Folder in Container` and select the repo directory.
   
3. Wait for the container to build and the dependencies to install
   
**Option 2: GitHub Codespaces**

1. Fork this [repo](https://github.com/MartyC-137/Data-Engineering-Devcontainer)
   
2. From your forked repo in GitHub, select the green `<> Code` button and choose Codespaces
   
3. Click `Create Codespace on Main`, or checkout a branch if you prefer
   
4. Wait for the container to build and the dependencies to install
   
5. Start developing!

## Components of a `devcontainer`

What is required for a `devcontainer`, and how does this work? When you have the [`Dev Containers`](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension installed and are setting up any new projects, VS Code will automatically detect if you have a `.devcontainer` directory inside your repo. The main file it is looking for is a `devcontainer.json` file, although you can augment this with a `Dockerfile`, a `requirements.txt` for Python libraries, etc. In this `devcontainer` you'll notice the following:

* `Dockerfile`
* `devcontainer.json`
* `config.fish`
* `Microsoft.Powershell_profile.ps1`

the bulk of the `Dockerfile`, `config.fish` and `Microsoft.Powershell_profile.ps1` are used to configure a custom powershell Powershell environment. Lately, I've been a big fan of the [Oh My Posh](https://github.com/JanDeDobbeleer/oh-my-posh) Powershell themes - this is an awesome repo and a great way to take your terminal experience to the next level! You can check out the official Microsoft documentation [here](https://learn.microsoft.com/en-us/windows/terminal/tutorials/custom-prompt-setup) as well for customizing your powershell experience.

Here's an example of what Oh My Posh looks like, I'm using the [clean-detailed theme](https://github.com/JanDeDobbeleer/oh-my-posh/blob/main/themes/clean-detailed.omp.json):

![image](/assets/images/oh-my-posh.png)