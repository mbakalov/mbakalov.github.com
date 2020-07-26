---
layout: post
title: "Running NUnit-based integration tests in parallel. Part 0: introduction and setup."
---

# Introduction

I have been exploring different options for running a large number of NUnit integration tests in parallel. There are several
constraints in my instance of this problem.

First, the tests use a database, and multiple tests cannot be run in parallel against the same database because they **don't
expect shared state**. This is not ideal and it basically forces us to have multiple copies of the db and also means we can't
use NUnit's `[Parallelizable]` easily. 

Second, all tests are in the **same DLL**. This is not as serious but means we can't make use of NUnit's
[Engine Parallel Test Execution](https://docs.nunit.org/articles/nunit/technical-notes/usage/Engine-Parallel-Test-Execution.html).

All the options I have been exploring are basically different takes on running multiple instances of the `nunit3-console.exe` process
and managing multiple databases for each of them. I'll write down different things that I tried in a series of posts:

* Part 0: this post. Initial setup, sample project with many generated tests and a basic TeamCity config.
* Part 1: running multiple parallel `nunit3-console.exe` processes on a single TC build agent and collecting their results.
* Part 2: running each set of tests in a separate container and using Azure Container Instances to deploy them.
* Part 3: try setting up multiple TeamCity build agents. From some googling it looks like this isn't easily possible, but maybe
  there is a nice solution with setting up multiple build configurations and merging their outputs somehow.

This post just covers the initial setup and doesn't deal with any parallelization yet! {% comment %}, skip to Part 1 for that. {% endcomment %}

# Setting Up Test Solution - Using T4 to Generate Test Code on Build

To experiment with tests I have a very small [.NET Framework solution](https://github.com/mbakalov/NUnit-Parallel-Examples/tree/master/src/FrameworkApp)
that has a simple User/Post/Comment model and uses Entity Framework to talk to the database.

There is a single `RunTest` method that creates a bunch of posts, users, and comments in the database, and then reads all of the DB data back to
verify there is nothing else there. To better simulate a real scenario, we want each test to run for at least a few seconds and we also want each test
to fail if *some other test* creates entries in the same tables.
```csharp
public static void RunTest(string fixtureName, string methodName)
{
    var db = new BlogContext();
    var testName = $"{fixtureName}.{methodName}";
    for (int i = 0; i < UserCount; i++)
    {
        var u = new User { Name = $"{testName} User {i}", CreatedOn = DateTime.Now };
        db.Users.Add(u);

        for (int j = 0; j < PostByEachUser; j++)
        {
            var p = new Post { Body = $"{testName} Post {j} by User {i}", CreatedOn = DateTime.Now, Owner = u };
            db.Posts.Add(p);
        }
    }

    db.SaveChanges();

    var users = db.Users.ToList();
    foreach (var p in db.Posts.ToList())
    {
        foreach (var u in users)
        {
            for (int k = 0; k < CommentByPost; k++)
            {
                var c = new Comment
                {
                    Author = u,
                    ParentPost = p,
                    CreatedOn = DateTime.Now,
                    Text = $"{testName}. Comment {k} on Post {p} by User {u}"
                };
                db.Comments.Add(c);
            }
        }
    }

    db.SaveChanges();

    Assert.AreEqual(UserCount * PostByEachUser * UserCount * CommentByPost, db.Comments.Count(), "Comment count");
}
```

Next, we want to be able to have many NUnit test methods and test fixtures, to simulate a real scenario when a single test DLL
contains a lot of them. One way to do this is to generate many classes and methods, with each method calling the same `RunTest` above.
This can be done e.g. by using [T4 templates](https://docs.microsoft.com/en-us/visualstudio/modeling/code-generation-and-t4-text-templates?view=vs-2019).
A template like this:
```csharp
<#@template language="c#" hostspecific="true"#>
<#@ parameter type="System.Int32" name="FixtureCount" #>
<#@ parameter type="System.Int32" name="TestsPerFixtureCount" #>
<#@ assembly name="System.Core" #>
<#@ import namespace="System.Linq" #>
<#@ import namespace="System.Text" #>
<#@ import namespace="System.Collections.Generic" #>

using System.Reflection;
using NUnit.Framework;

namespace FrameworkApp.Tests.Generated
{
<#
for (int i = 1; i <= FixtureCount; i++) { #>
	public class Fixture<#=i #>: BaseFixture
	{
	<#
	for (int j = 1; j <= TestsPerFixtureCount; j ++) { #>
	[Test]
		public void Test<#=j #>()
		{
			TestHelper.RunTest(GetType().Name, MethodBase.GetCurrentMethod().Name);
		}
	<# } #>
}
<# } #>
}
```

Will generate a single file containing code like:
```csharp
public class Fixture1: BaseFixture
{
    [Test]
    public void Test1()
    {
        TestHelper.RunTest(GetType().Name, MethodBase.GetCurrentMethod().Name);
    }
    [Test]
    public void Test2()
    {
        TestHelper.RunTest(GetType().Name, MethodBase.GetCurrentMethod().Name);
    }
    // etc
}
public class Fixture2: BaseFixture
{
    [Test]
    public void Test1()
    {
        TestHelper.RunTest(GetType().Name, MethodBase.GetCurrentMethod().Name);
    }
    // etc
}
```

We also want to be able to configure how many tests should be generated during our build. T4 supports passing parameters
to templates like below:
```csharp
<#@template language="c#" hostspecific="true"#>
<#@ parameter type="System.Int32" name="FixtureCount" #>
<#@ parameter type="System.Int32" name="TestsPerFixtureCount" #>
```

Set up this way, the parameters will both be zero when building in Visual Studio, but all we need is to be able to
pass them in via MSBuild, which works:
```xml
<PropertyGroup>
    <TransformOnBuild>true</TransformOnBuild>
    <OverwriteReadOnlyOutputFiles>true</OverwriteReadOnlyOutputFiles>
    <TransformOutOfDateOnly>false</TransformOutOfDateOnly>
    <FixtureCount>10</FixtureCount>
    <TestsPerFixtureCount>10</TestsPerFixtureCount>
</PropertyGroup>
<ItemGroup>
    <T4ParameterValues Include="FixtureCount">
        <Value>$(FixtureCount)</Value>
        <Visible>false</Visible>
    </T4ParameterValues>
    <T4ParameterValues Include="TestsPerFixtureCount">
        <Value>$(TestsPerFixtureCount)</Value>
        <Visible>false</Visible>
    </T4ParameterValues>
</ItemGroup>
<ItemGroup>
    <!-- Skip some files -->
    <Compile Include="Generated\GenerateFixtures.cs">
        <AutoGen>True</AutoGen>
        <DesignTime>True</DesignTime>
        <DependentUpon>GenerateFixtures.tt</DependentUpon>
    </Compile>
</ItemGroup>
```

# Configuring TeamCity Build Agent - Installing Build Tools and SQL Server

Ultimately we want our builds to be triggered in TeamCity and we want to see real-time test results in there as well. In each
post I'll describe additional TC config settings and tweaks I was making along the way.

I started with a clean default TC install on a [Windows Server 2019 Azure VM]({% post_url 2020-07-19-windows-server-2019-sql-docker-azure %}).
Maintaining a build server without Visual Studio on it used to be quite difficult years ago, but nowadays simply installing
[Visual Studio Build Tools](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2019) with appropriate components
is almost always enough.

I did have to copy over some additional bits from my development machine to get T4 to work on the build server, but there is a
[Mircosoft Docs section](https://docs.microsoft.com/en-us/visualstudio/modeling/code-generation-in-a-build-process?view=vs-2019) that describes
exactly what is needed.

Finally, within TeamCity I have installed NuGet and NUnit tools via its [UI](https://www.jetbrains.com/help/teamcity/installing-agent-tools.html)
and also enabled Kotlin DSL, to be able to modify build settings directly in my project's GitHub [repo](https://github.com/mbakalov/NUnit-Parallel-Examples/tree/master/.teamcity).
Kotlin DSL was easy to enable on a new project and I pretty much just had to follow the instructions in JetBrains' [docs](https://www.jetbrains.com/help/teamcity/kotlin-dsl.html).

My initial version of `settings.kts` that just builds the project and runs NUnit tests *without any parallelism **yet*** looks like this:
```kotlin
import jetbrains.buildServer.configs.kotlin.v2019_2.*
import jetbrains.buildServer.configs.kotlin.v2019_2.buildSteps.*
import jetbrains.buildServer.configs.kotlin.v2019_2.triggers.vcs

version = "2020.1"

project {

    buildType(Build)
}

object Build : BuildType({
    name = "Build"

    vcs {
        root(DslContext.settingsRoot)
    }

    steps {
        nuGetInstaller {
            toolPath = "%teamcity.tool.NuGet.CommandLine.DEFAULT%"
            projects = "src/FrameworkApp/FrameworkApp.sln"
        }
        dotnetMsBuild {
            projects = "src/FrameworkApp/FrameworkApp.sln"
            version = DotnetMsBuildStep.MSBuildVersion.V16
            args = "-restore -noLogo"
            param("dotNetCoverage.dotCover.home.path", "%teamcity.tool.JetBrains.dotCover.CommandLineTools.DEFAULT%")
        }
        powerShell {
            name = "Setup SQL Server in a container to run tests against"
            scriptMode = file {
                path = ".teamcity/Add-SQLServer.ps1"
            }
        }
        nunit {
            name = "Run integration tests"
            nunitPath = "%teamcity.tool.NUnit.Console.DEFAULT%"
            includeTests = """src\FrameworkApp\FrameworkApp.Tests\bin\Debug\FrameworkApp.Tests.dll"""
        }
        powerShell {
            name = "Tear down SQL Server container"
            scriptMode = file {
                path = ".teamcity/Remove-SQLServer.ps1"
            }
        }
    }

    triggers {
        vcs {
        }
    }
})
```

An interesting step here is the PowerShell `Add-SQLServer.ps1`. The tests need a database and I wanted to avoid installing
a full-blown SQL Server instance on the build agent (or somewhere else), so I tried running SQL in a Windows container on the build agent
host instead. I have a few more details about that set up in a [separate post]({% post_url 2020-07-19-windows-server-2019-sql-docker-azure %}), but the
script looks like this:
```powershell
$saPassword = "h4rdc0dedThr0wAw4yPwd!"
& docker pull microsoft/mssql-server-windows-developer
& docker run `
    --name tcdemo-sql-001 `
    -e "ACCEPT_EULA=Y" `
    -e "SA_PASSWORD=$saPassword" `
    -p 1444:1433 -d --isolation=hyperv `
    microsoft/mssql-server-windows-developer

# Create a separate SQL user to use in the connection string
# for the test DLL app.config.
# There is no good way to know when it is safe to execute an
# SQL command on container start, so we must try several times.
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

So on each build after compiling the code we will start a new instance of SQL Server in a container, create a test user,
then run all our tests, and finally stop/remove the container in the end.

The `app.config` for the [test DLL](https://github.com/mbakalov/NUnit-Parallel-Examples/tree/master/src/FrameworkApp/FrameworkApp.Tests)
references the DB running in the container:
```xml
<connectionStrings>
    <add
        name="BlogContext"
        connectionString="Data Source=localhost,1444;Initial Catalog=BlogDBTest;user id=testuser;password=testpassword"
        providerName="System.Data.SqlClient"/>
</connectionStrings>
```

And the `Remove-SQLServer.ps1` simply does:
```powershell
& docker stop tcdemo-sql-001
& docker rm tcdemo-sql-001
```

And that's it! With this we have a basic TeamCity setup with an ability to run a number of integration tests against a database, but
without any parallelism yet. In the next part{% comment %}[next part]({% post_url 2020-07-26-nunit-integration-parallel-teamcity-part1 %}){% endcomment %} we'll introduce
multiple NUnit processes and see how that can be made to work in TeamCity.

As always, hopefully this can be helpful to someone! All PowerShell, C#, and Kotlin ;) example code is in [this GitHub repo](https://github.com/mbakalov/NUnit-Parallel-Examples).

Cheers!