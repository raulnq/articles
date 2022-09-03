## Rundeck: Operational tasks Automation

Once our software is released to production, we enter a new stage of the software development life cycle, Operations and Maintenance:

> The maintenance stage is the final, and continuous, stage of iterating and building upon your software solution as it operates and progresses in a production environment. This could include bug fixes, upgrading security protocols, updating features and specifications, among many others.

This stage involves several tasks that require people, knowledge, and time. Hopefully, in many cases, those tasks can be automated (or part of them), and here is where [Rundeck](https://www.rundeck.com/) comes to the rescue:

> Rundeck is runbook automation that gives you and your colleagues self-service access to the processes and tools they need to get their job done.

> When used for incident management, Rundeck will help you have shorter incidents and fewer escalations.

> When used for general operations work, Rundeck will help alleviate the time-consuming and repetitive toil that currently consumes too much of your team's time.

In Rundeck, you define workflows (called [Jobs](https://docs.rundeck.com/docs/manual/04-jobs.html#overview)) from any of your existing tools or scripts and trigger those from Web UI, API, CLI, or by schedule. To understand the idea, we will set up a job to restart an Azure Web App every day at night.

### Pre-requisites

- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- [Azure Account](https://azure.microsoft.com/en-us/free/)

Let's start creating the Azure Web App (run the commands line by line):

```powershell
az login
az group create -l eastus -n MyResourceGroup
az appservice plan create -g MyResourceGroup -n MyPlan --sku B1
az webapp create -n "MyWebApp-24f12674-a141-43d9-a29d-9eb3e7bc5aca" -g MyResourceGroup -p MyPlan -r "dotnet:6"
``` 

Remember that the Azure Web App name must be unique, change it if you get an error. Automated tools that use Azure services should have restricted permissions. For this reason, Azure offers the `service principal` concept. Run `az account show --query id` command and replace `<SUBSCRIPTION_ID>` with the result in the following command (save the results in a safe place):

```powershell
az ad sp create-for-rbac --scopes /subscriptions/<SUBSCRIPTION_ID> --role "contributor"
``` 

### Rundeck

Run the following command to start Rundeck:

```powershell
docker run --name local-rundeck -p 4440:4440 -d rundeck/rundeck:4.5.0
```

Let's go inside the container:

```powershell
docker exec -it local-rundeck bash
``` 

Once in the container, we will proceed to install the Azure CLI (this could take a couple of minutes):

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
``` 

Let's open `http://127.0.0.1:4440/user/login` and use as username and password the word `admin`:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662220364885/NPM_Fp9OD.png align="left")

Create a [Project](https://docs.rundeck.com/docs/manual/projects/):

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662220555439/PPjkQycdH.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662220618277/PODmij-Kf.png align="left")

Go to the Jobs menu and create a new one:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662220701214/WZgSoyByX.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662220924655/dFUoGOvQC.png align="left")

Go to the [Workflow](https://docs.rundeck.com/docs/manual/job-workflows.html#workflow-control-settings) tab (here is where we define what our job will do):

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662221403029/Z-MtugsRw.png align="left")

In our case, we are going to create two steps (command type), one to login against Azure:

```bash
az login --service-principal -u <SERVICE_PRINCIPAL_APPID> -p <SERVICE_PRINCIPAL_PASSWORD> --tenant <SERVICE_PRINCIPAL_TENANT>
``` 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662221920686/VtD3I19hT.png align="left")

And the second to execute the restart itself:

```bash
az webapp restart -n MyWebApp-24f12674-a141-43d9-a29d-9eb3e7bc5aca -g MyResourceGroup
```

![image (1).png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662222088366/hEvlZSxT0.png align="left")

Go to the Schedule tab to set up when is going to run:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662222253611/7vYRvVuV_.png align="left")

Let's test our first job by running it manually through the web UI:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662222573767/D8UMQ1GPO.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662222623815/s-MqeU5GJ.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662222651092/-q5Hh4DcP.png align="left")

Minutes later, we can check the Activity Log on Azure to see the restart:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662224423547/hHFkRgcqS.png align="left")

[Here](https://docs.rundeck.com/docs/), you can find the official documentation. Thanks, and happy coding. 
