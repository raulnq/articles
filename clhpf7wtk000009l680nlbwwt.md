---
title: "How to Deploy a Windows Service on AWS Elastic Beanstalk using Terraform"
datePublished: Mon May 15 2023 22:32:43 GMT+0000 (Coordinated Universal Time)
cuid: clhpf7wtk000009l680nlbwwt
slug: how-to-deploy-a-windows-service-on-aws-elastic-beanstalk-using-terraform
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1684070174050/4e0f0dec-0ccd-4d48-aae1-eee1606e7f76.png
tags: aws, net, elastic-beanstalk, windows-service

---

Hosting our applications on Windows Servers will lead us to consider using Windows Services for running background processes or applications without user interaction. In a previous post, we learned [How to Deploy a .NET App on AWS Elastic Beanstalk using Terraform](https://blog.raulnq.com/how-to-deploy-a-net-app-on-aws-elastic-beanstalk-using-terraform-windows-server). Today we'll expand this deployment to include a Windows Service.

So, download or clone [this](https://github.com/raulnq/aws-beanstalk/tree/windows) repository. Before adding the new project, we will move the `WeatherAPI` project to its folder under `src`. Next, we will run the following command:

```bash
dotnet new worker --name WeatherWs -o src/WeatherWs
dotnet sln add --in-root src/WeatherWs
dotnet add src/WeatherWs package Microsoft.Extensions.Hosting.WindowsServices
```

Open the `Program.cs` file and replace the content with:

```bash
using WeatherWs;

IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddHostedService<Worker>();
    })
    .UseWindowsService()
    .Build();

host.Run();
```

The Windows Service is ready. Create an `install.ps1` file at the solution level with the following content:

```powershell
$serviceName="WeatherWs"
$serviceFolder="C:\services\WeatherWs"
$exe="$serviceFolder\WeatherWs.exe" 
$bin="$PSScriptRoot\ws"
mkdir $serviceFolder -Force
$exists = Get-WmiObject -Class Win32_Service -Filter "Name='$serviceName'"
if($exists)
{
	Stop-Service -Name $serviceName -Force 
	Start-Sleep -s 5
	sc.exe delete $serviceName
	Start-Sleep -s 5
}
Copy-Item "$bin\*" $serviceFolder -Recurse -Force
New-Service -Name $serviceName -BinaryPathName $exe
Start-Service -Name $serviceName
```

> PSScriptRoot: Contains the full path to the script that invoked the current command. The value of this property is populated only when the caller is a script.

Modify the deployment manifest (`aws-windows-deployment-manifest.json`) with the following content:

```json
{
    "manifestVersion": 1,
    "deployments": {
        "aspNetCoreWeb": [
        {
            "name": "dotnet-api",
            "parameters": {
              "appBundle": "site.zip",
              "iisPath": "/",
              "iisWebSite": "Default Web Site"
            },
            "scripts": {
              "preInstall": {
                "file": "install.ps1"
              }
          }
        }
        ]
    }
}
```

With this configuration, we'll run our script before the deploy the API. Further information about the deployment manifest can be found [here](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/dotnet-manifest.html). Now, the most important part, the bundle creation:

* Publish the API in a folder.
    
* Zip the published API and copy the file to the bundle folder.
    
* Publish the Windows Service directly in the bundle folder.
    
* Copy the `install.ps1` and the `aws-windows-deployment-manifest.json` to the bundle folder
    
* Zip the bundle folder.
    

```bash
dotnet publish ./src/WeatherApi/WeatherApi.csproj --output "terraform/publish/webapi" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime win-x64 --no-self-contained

Compress-Archive -Path terraform/publish/webapi/* -DestinationPath terraform/bundle/site.zip

dotnet publish ./src/WeatherWs/WeatherWs.csproj --output "terraform/bundle/ws" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime win-x64 --no-self-contained

copy .\install.ps1 .\terraform\bundle
copy .\aws-windows-deployment-manifest.json .\terraform\bundle

Compress-Archive -Path terraform/bundle/* -DestinationPath terraform/app.zip
```

The contents of the bundle (`app.zip`) will appear as follows:

```bash
|-- ws
|-- aws-windows-deployment-manifest.json
|-- install.ps1
`-- site.zip
```

Run the terraform scripts with the following commands:

```bash
cd terraform
terraform init
terraform plan -out app.tfplan -var="health_check_path=/swagger/index.html" -var="bucket=app-tf-001" -var="keypair=<MY_KEY_PAIR>" -var="instance_type=t2.medium" -var="application=app-tf-001" -var="vpc_id=<MY_VPC>" -var="ec2_subnets=<MY_SUBNETS>" -var="elb_subnets=<MY_SUBNETS>" -var="platform=64bit Windows Server 2019 v2.11.3 running IIS 10.0"
terraform apply 'app.tfplan'
```

And deploy the application version with the following command:

```bash
aws --region us-east-2 elasticbeanstalk update-environment --environment-name <OUTPUT_ENV_NAME> --version-label <OUTPUT_APP_VERSION>
```

And that's it. There is an alternative method for deploying Windows Services using `.ebextensions` (you can check the [video tutorial](https://www.youtube.com/watch?v=hzF3E30-Jd8) and find further information [here](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/ebextensions.html)), but we believe this approach is simpler. The code and scripts are available [here](https://github.com/raulnq/aws-beanstalk/tree/windows-service). Thanks, and happy coding.