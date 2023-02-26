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
## Getting Started with Dev Containers

Recently I discovered [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers), which allows you to open a directory inside of a docker container and use it as a complete development environment in [Visual Studio Code](https://code.visualstudio.com/) or [GitHub Codespaces](https://github.com/features/codespaces). Configuring an environment and installing all of the required dependencies for a project can be exceedingly difficult, so I thought it would be great to set up a reproducible environment for my team to use at work. 

Specifically, setting up [`Snowpark`](https://docs.snowflake.com/en/developer-guide/snowpark/python/setup) for Snowflake peaked my interest in Dev Containers - it requires Python 3.8 anaconda, and the [Snowpark Python Package](https://pypi.org/project/snowflake-snowpark-python/). Its fairly straightforward to set the above environment up, but after you do it a few times, you start to wonder if theres a better way. Turns out there is :smiley: Any time you start a new project, you can simply copy over your devcontainer to configure your environment.

This Dev Container has a data engineering flavour - It likely won't suit all of your needs but hopefully it will be a great starting point for you.

Dev Containers require you to have [Docker](https://www.docker.com/products/docker-desktop/) installed OR use [GitHub Codespaces](https://github.com/features/codespaces). Codespaces is now free for individual use (60 hours/month) and is worth checking out if you haven't tried it.

**Option 1: Local VS Code**

1. Clone the [repo](https://github.com/MartyC-137/Data-Engineering-Devcontainer) and connect to it in VS Code:

```sh
$ cd your/desired/repo/location
$ git clone https://github.com/MartyC-137/DataEng_devcontainer
```

1. Download the [`Dev Containers`](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension from the VS Code marketplace

2. Press Cmd + Shift + P (Mac) or Ctrl + Shift + P (Windows) to open the Command Pallette. Type in `Dev Containers: Open Folder in Container` and select the repo directory.
   
3. Wait for the container to build and the dependencies to install
   
**Option 2: GitHub Codespaces**

1. [`Fork` this repo](https://github.com/MartyC-137/Data-Engineering-Devcontainer)
   
2. From your forked repo in GitHub, select the green `<> Code` button and choose Codespaces
   
3. Click `Create Codespace on Main`, or checkout a branch if you prefer
   
4. Wait for the container to build and the dependencies to install
   
5. Start developing!

## Components of a Dev Container

When you have the [`Dev Containers`](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension installed and are setting up a new project, VS Code and GitHub Codespaces will automatically detect if you have a `.devcontainer` directory inside your repo. It is looking for is a `devcontainer.json` file, although you can augment this with a `Dockerfile`, a `requirements.txt` etc. In the `.devcontainer` directory from my [repo](https://github.com/MartyC-137/Data-Engineering-Devcontainer) you'll see the following:

* `Dockerfile`
* `devcontainer.json`
* `config.fish`
* `Microsoft.Powershell_profile.ps1`
* `reuquirements.txt`

the bulk of the `Dockerfile`, `config.fish` and `Microsoft.Powershell_profile.ps1` are used to configure a custom powershell Powershell environment, [Oh My Posh](https://github.com/JanDeDobbeleer/oh-my-posh). This application provides awesome Powershell themes and is a great way to take your terminal experience to the next level. You can check out the official Microsoft documentation [here](https://learn.microsoft.com/en-us/windows/terminal/tutorials/custom-prompt-setup). I really love using the terminal now since starting to use Oh My Posh.

Here's an example of what Oh My Posh looks like, here I'm using the [clean-detailed theme](https://github.com/JanDeDobbeleer/oh-my-posh/blob/main/themes/clean-detailed.omp.json):

![image](/assets/images/devcontainer_blog/oh-my-posh.png)

The `Microsoft.Powershell_profile.ps1` is a configuration file that runs when Powershell starts. Please note that I copied this template from the [Oh My Posh repo](https://github.com/JanDeDobbeleer/oh-my-posh/blob/main/.devcontainer/Microsoft.PowerShell_profile.ps1) - you can add anything you like here to run when Powershell starts up. In mine, I've added one alias that seems to not work on Mac, but you can add anything you'd like to run here when Powershell starts.

The [Terminal Icons](https://github.com/devblackops/Terminal-Icons) theme is included in the Powershell profile - if you run the [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem?view=powershell-7.3) cmdlet inside of your devcontainer, you'll see the addition of icons for files:

![image](/assets/images/devcontainer_blog/terminal-icons.png)
<!-- <img src="/assets/images/devcontainer_blog/terminal-icons.png" width="600" height="400"> -->
Next, lets take a look at the `devcontainer.json` file. Here is what ours looks like for this project:

```json
{
    "name": "oh-my-posh",
	"build": {
        "dockerfile": "Dockerfile",
	  "args": {
          "VARIANT": "1.19-bullseye",
		"POSH_THEME": "https://raw.githubusercontent.com/JanDeDobbeleer/oh-my-posh/main/themes/clean-detailed.omp.json",

		"TZ": "America/Moncton",
  
		"NODE_VERSION": "lts/*",
		"PS_VERSION": "7.2.7"
	  }
	},
	"runArgs": ["--cap-add=SYS_PTRACE", "--security-opt", "seccomp=unconfined"],

	"features": {
        "ghcr.io/devcontainers/features/azure-cli:1": {
            "version": "latest"
		},
		"ghcr.io/devcontainers/features/python:1": {
            "version": "3.8"
		},
		"ghcr.io/devcontainers-contrib/features/curl-apt-get:1": {},
		"ghcr.io/devcontainers-contrib/features/terraform-asdf:2": {},
		"ghcr.io/devcontainers-contrib/features/yamllint:2": {},
		"ghcr.io/devcontainers/features/docker-in-docker:2": {},
		"ghcr.io/devcontainers/features/docker-outside-of-docker:1": {},
		"ghcr.io/devcontainers/features/github-cli:1": {},
		"ghcr.io/devcontainers-contrib/features/spark-sdkman:2": {
            "jdkVersion": "11"
		},
		"ghcr.io/dhoeric/features/google-cloud-cli:1": {
            "version": "latest"
		}
	  },

	"customizations": {
        "vscode": {
            "settings": {
                "go.toolsManagement.checkForUpdates": "local",
			"go.useLanguageServer": true,
			"go.gopath": "/go",
			"go.goroot": "/usr/local/go",
			"terminal.integrated.profiles.linux": {
                "bash": {
                    "path": "bash"
				},
				"zsh": {
                    "path": "zsh"
				},
				"fish": {
                    "path": "fish"
				},
				"tmux": {
                    "path": "tmux",
				"icon": "terminal-tmux"
				},
				"pwsh": {
                    "path": "pwsh",
				"icon": "terminal-powershell"
				}
			},
			"terminal.integrated.defaultProfile.linux": "pwsh",
			"terminal.integrated.defaultProfile.windows": "pwsh",
			"terminal.integrated.defaultProfile.osx": "pwsh",
			"tasks.statusbar.default.hide": true,
			"terminal.integrated.tabs.defaultIcon": "terminal-powershell",
			"terminal.integrated.tabs.defaultColor": "terminal.ansiBlue",
			"workbench.colorTheme": "GitHub Dark Dimmed",
			"workbench.iconTheme": "material-icon-theme"
			},

			"extensions": [
                "ms-mssql.mssql",
				"snowflake.snowflake-vsc",
				"golang.go",
				"ms-vscode.powershell",
				"ms-python.python",
				"ms-python.vscode-pylance",
				"redhat.vscode-yaml",
				"ms-vscode-remote.remote-containers",
				"ms-toolsai.jupyter",
				"eamodio.gitlens",
				"yzhang.markdown-all-in-one",
				"davidanson.vscode-markdownlint",
				"editorconfig.editorconfig",
				"esbenp.prettier-vscode",
				"github.vscode-pull-request-github",
				"akamud.vscode-theme-onedark",
				"PKief.material-icon-theme",
				"GitHub.github-vscode-theme",
				"actboy168.tasks",
				"bastienboutonnet.vscode-dbt",
				"innoverio.vscode-dbt-power-user"
			]
		}
	},
  
	// Use 'forwardPorts' to make a list of ports inside the container available locally.
	// "forwardPorts": [3000],
  
	"postCreateCommand": "pip3 install --user -r .devcontainer/requirements.txt --use-pep517",

	"remoteUser": "vscode"
  }
```

This file has a few key components:

* `build` block - Specifies that we are using a `Dockerfile`. The `Dockerfile` uses a `Go` base image because Oh My Posh is written in Go. I install Python later
  
* [`features`](https://code.visualstudio.com/blogs/2022/09/15/dev-container-features) block - this is an easy way to install additional programming languages, command line tools etc into your devcontainer. In my devcontainer I have the following:
    - `Python 3.8`
    - `Terraform`
    - `Azure CLI`
    - `Google Cloud CLI`
    - `GitHub CLI`
    - `Spark` with JDK 11
  
* `customizations` block - this passes default values to our devcontainers `setings.json` file, the file that configures our VS Code settings. Here I set values like Powershell as the default terminal, GitHub Dark Dimmed, Material Icons theme etc. These are my personal preferences, please update these with anything else you prefer. **Note that Oh My Posh does work with bash, zsh etc.** You don't have to use Powershell if you prefer a different shell!
  
* `extensions` block - this block installs any VS Code extensions you like. In this devcontainer I've included the following:
    - `SQL Server` - note that you'll have to forward the correct ports for this to connect to an on-premise database from your container
    - `Snowflake`
    - `Powershell`
    - `Python` tools
    - `Jupyter Notebooks`
    - GitHub Pull Requests
    - GitLens
    - Popular VS Code Themes (GitHub, One Dark etc.)
    - `dbt` extensions
* `postCreateCommand` - this line instructs our devcontainer to run the included `requirements.txt` file to `pip` install the following packages:

    - `pandas`
    - `prefect` and its various dependencies
    - `sqlalchemy`
    - `ipykernel`
    - `polars`
    - `dbt-core`
    - `dbt-bigquery`
    - `dbt-snowflake`
    - `dbt-postgres`
    - `pyspark`
    - `confluent-kafka`
    - `snowpark`
    - `scikit-learn`

And there you have it! You've now got an awesome development environment set up. Happy Coding!

<!-- ![image](/assets/images/devcontainer_blog/vs-code_dc.png) -->
<img src="/assets/images/devcontainer_blog/vs-code_dc.png" width="800" height="800">