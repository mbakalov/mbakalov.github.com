---
layout: post
title: "Running a SQL Server (Windows) Docker image on a Windows Server 2019 host in Azure"
---

## Introduction

For some experiments in setting up parallel execution of NUnit tests that use a SQL Server database ([series here]({% post_url 2020-07-26-nunit-integration-parallel-teamcity-part0 %})),
one of the things I wanted to try is a scenario when the SQL Server is deployed in a container image with the host being the TeamCity build agent itself.

I wanted to avoid doing a full SQL Server install on the agent (or elsewhere) and thought a container would be an interesting option to try. Overall it turned out to be a bit
tricky to configure and ended up being a bit unstable, so I wouldn't use that in a "real" CI pipeline (and definitely not in production!), but still an interesting experience
that can hopefully be useful to someone.

The image I used is [microsoft/mssql-server-windows-developer](https://hub.docker.com/r/microsoft/mssql-server-windows-developer/).
Unlike the [Linux version](https://hub.docker.com/_/microsoft-mssql-server), the Windows image hasn't been updated in several years, first sign that this
wasn't going to be a smooth setup. It looks like it is built from [this GitHub repo](https://github.com/microsoft/mssql-docker/tree/master/windows),
so it would be interesting to try to rebuild it using a newer Windows Server Core base image, for example.

## Create a Windows Server 2019 VM
Starting with a new Azure VM that will be the TeamCity build agent and the container host:
```powershell
$rg = "rg-tcdemo-vsprof-001"
$vm = "vmteamcity001"
$nsg = $vm + "nsg"
$image = "Win2019Datacenter" # Using latest Windows Server 2019

# Not all sizes will work running the required container.
# Use something that supports "nested virtualization" (see below).
$size = "Standard_D2s_v3" 

& az group create -n $rg -l northcentralus

$admin = Get-Credential -Message "Enter new VM admin user credentials"
& az vm create `
    --resource-group $rg `
    --name $vm `
    --image $image `
    --admin-username $admin.GetNetworkCredential().UserName `
    --admin-password $admin.GetNetworkCredential().Password `
    --size $size
```

Here a few things are important. First of all, assuming we have Windows Server 2019 as our OS,
we are going to run into trouble later when trying to run the `microsoft/mssql-server-windows-developer` image, because it is based on
Windows Server 2016. To run an older OS kernel version we need to use [Hyper-V isolation](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container),
so we need to install Docker EE first (obviously) and then enable Hyper-V on the VM.

## Install Docker and Enable Hyper-V
```powershell
Install-Module -Name DockerMsftProvider -Repository PSGallery -Force
Install-Package -Name docker -ProviderName DockerMsftProvider
Restart-Computer -Force

Enable-WindowsOptionalFeature –Online -FeatureName Microsoft-Hyper-V –All -NoRestart
Install-WindowsFeature RSAT-Hyper-V-Tools -IncludeAllSubFeature
Restart-Computer -Force
```

Another important thing here is that since we are going to use Hyper-V, and our host is an Azure VM, using Hyper-V on it requires SKUs
that support [Nested Virtualization](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/nested-virtualization). The `Standard_D2s_v3` is
one such size, but others can be found [here](https://docs.microsoft.com/en-us/azure/virtual-machines/acu) (marked with the "***").

## Start the Container and Execute Some Scripts
Once the host is properly configured, we can finally pull and start the image:
```powershell
$saPassword = "h4rdc0dedThr0wAw4yPwd!"
& docker pull microsoft/mssql-server-windows-developer
& docker run `
    --name tcdemo-sql-001 `
    -e "ACCEPT_EULA=Y" `
    -e "SA_PASSWORD=$saPassword" `
    -p 1433:1433 -d --isolation=hyperv `
    microsoft/mssql-server-windows-developer
```

Note the `--isolation=hyperv` above.

Once the image is started we typically want to execute some custom scripts to prep the SQL server for doing some useful work.
In my case I needed to create test SQL users, but another common scenario is seeding a test database with data. Surprisingly,
there is no "nice" way to do this ([GitHub issue](https://github.com/microsoft/mssql-docker/issues/11)).

The GitHub issue above lists a few workarounds, one of which is to keep retrying the startup command until it succeeds, which
is what I ended up doing:

```powershell
for ($i = 0; $i -lt 20; $i++) {
    & docker exec tcdemo-sql-001 sqlcmd `
        -S localhost -U sa -P "$saPassword" `
        -Q "CREATE LOGIN [testuser] WITH PASSWORD=N'testpassword', DEFAULT_DATABASE=[master], CHECK_EXPIRATION=OFF, CHECK_POLICY=OFF"

    if ($LASTEXITCODE -eq 0) {
        Write-Host "testuser created"
        break
    } else {
        Write-Host "Failed to create testuser, probably SQL Server hasn't started yet. Will retry in 15 seconds. Attempt $i/20"
        Start-Sleep -Seconds 15
    }
}

& docker exec tcdemo-sql-001 sqlcmd `
    -S localhost -U sa -P "$saPassword" `
    -Q "ALTER SERVER ROLE [sysadmin] ADD MEMBER [testuser]"
```

And that is pretty much it. Hope this helps someone out there!

## Gotchas
Here are a few more details about the errors I encountered while trying to do this. Hopefully they could be useful
to someone googling for the error messages.

### Windows Server Version
While trying to run the `microsoft/mssql-server-windows-developer` image you may get an error like:

```
docker.exe: Error response from daemon: hcsshim::CreateComputeSystem
The container operating system does not match the host operating system.
```

This happens because the host OS has a newer kernel (Windows Server 2019 in my case) while the container
OS is based on Windows Server 2016.

The solution for this is to enable Hyper-V on the host and to run the container using `--isolation=hyperv`.

### Nested Virtualization
While trying to run the image using `--isolation=hyperv` option you may get an error like:
```
docker.exe: Error response from daemon: hcsshim::CreateComputeSystem
The request is not supported.
```

In my case that was because I didn't actually enable Hyper-V. Once it is "enabled", you may still get an error:

```
docker.exe: Error response from daemon: hcsshim::CreateComputeSystem 
The virtual machine could not be started because a required feature is not installed.
```

In my case this was because I used an Azure VM size that didn't support [Nested Virtualization](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/nested-virtualization),
even though the "Hyper-V" Windows feature was highlighted as "enabled" in the UI (and I don't think I saw any errors when trying to enable it via Powershell in the first place).

I had initially started with the `Standard_B2ms` size, which didn't work. Once I resized to `Standard_D2s_v3` I was able to finally start the container.

### Container Hangs on `docker run`
For this one I haven't been able to find the solution unfortunately. I have hit a scenario a few times where `docker run` would just hang trying to start the container.
The container would be listed by `docker ps` but trying to `stop` or `rm -f` it would also just hang.

The only way I could remove the container is by restarting the docker service first. The problem seems to be similar to [this issue](https://github.com/docker/for-win/issues/4554),
which is closed as stale without a good solution :(
