--- 
layout: post 
title:  "Renew your builds - A piece of cake" 
description: Bring your build environment up to date using the Cake (C# Make) build system and get a tool for handling your solutions on top.
categories: build-automation development cake .net 
--- 

At the beginning of the year we moved our SVN repository to git and introduced
the **T**eam **F**oundation **S**erver to our development team. After the switch
we planned our next steps and decided that we wanted to try the TFS build system. Doing that we wanted to give the Cake build system a try. 

<blockquote cite="https://cakebuild.net/">
Cake (C# Make) is a cross platform build automation system with a C# DSL to do things like compiling code, copy files/folders, running unit tests, compress files and build NuGet packages.
</blockquote>

Let me sketch out our situation a little bit and how we integrated Cake.

# XMHell
Did I spell that correctly? My bad! But seriously, is there anyone who actually likes editing XML files by hand? I refuse to believe that. Our current build files are MsBuild XML files that define what's part of the build process. Some of them more complex than others. Especially if you call some external tools that need escaping of characters that are used by XML then you end up with stuff like 
```xml
<SomeElement attribute="&quot;&amp;canyoustillreadthis?&quot;">
```
 Do I have to say more? 

 With Cake you can write nice and clean C# syntax, use methods and classes to structure your code and use the .NET Framework classes 
and helpers. On top of that Cake offers many extensions that help with managing tasks, arguments or tools. See this minimal example from the Cake homepage:

 ```csharp
var target = Argument("target", "Default");

Task("Default")
  .Does(() =>
  {
    Information("Hello World!");
  });

RunTarget(target);
 ```
Isn't that more convenient than XML? And if your build files get bigger, you can split them up in several files and include them with the `#load file.cake` directive.

# Bob our special snowflake
We have a build server with the same name as the guy with the motto "Yes we can". No, not Obama. It's Bob, Bob the builder! It is a Windows machine that runs Jenkins and several slave processes. It's responsible for building the releases of our internally used software suite and for triggering the deployments. It also performs continuous and daily builds and gives feedback to developers. I really like Jenkins and I think it does what it does pretty well. I was quite happy working with it - yes, also for .NET projects. The reasons for switching to TFS shall not be part of this post.
{% include image.html name="yes_we_can.png" caption="Bob the builder - yes we can!" class="image-float-right" %}
Because the build machine already has a few years on its back and has to build a lot of different projects the requirements to the installed software grew over time. Different frameworks, ui components, test tools, documentation generators, compiler versions code analysis tools are just a few among others. Each of which also have multiple versions available. This makes our "Bobby" a [very special snowflake](https://martinfowler.com/bliki/SnowflakeServer.html){:target="_blank"}. And everyone that had to reproduce an error that just occurs on the build machine knows the pain something like this can cause. 

Of course if we had treated the TFS environment the same way it would also be a mess, but we had the chance to create it from scratch. This means we could install some close to vanilla Windows machines with TFS build agents and the [Visual Studio 2017 build tools](https://www.visualstudio.com/de/downloads/){:target="_blank"}. This is pretty much everything that is installed.

We still need a test runner, and all those other things that were previously installed on the build machine. So what are we going to do? Well, there is nothing that couldn't be solved with cake. :) The built in support that Cake offers for handling tools and other dependencies is great for that. It uses NuGet packages so you can use every tool or library that is available via NuGet package within your build script. It also has very comfortable extension methods for the commonly used ones like NUnit, MsBuild and [many more](https://cakebuild.net/dsl/){:target="_blank"}.

So what we do now instead of installing NUnit in several versions, we just specify the NUnit tool and its version in the projects build script like so and it is loaded and used by Cake. 

```csharp
#tool "nuget:?package=NUnit.ConsoleRunner"

NUnit3("./tests/**/*.Tests.dll", new NUnit3Settings {
  NoResults = true
  // other settings
});
```

The same goes for static code analysis, code coverage, running database scripts, powershell and so on. There is a whole lot built in functionality. And if it's not, there are ...

# Addins 
{% include image.html name="cake-contrib.png" caption="Cake contrib" class="image-float-left" %}If the need for a tool occurs that does not have built in C# wrappers for easier handling, chances are that somebody else needed that too and wrote an addin for it. There are already a lot of them available [as you can see on the homepage](https://cakebuild.net/addins/){:target="_blank"}. If you can't find it, there is always the chance to write your own addin. I wanted to run liquibase database migration scripts and created the [Cake.Liquibase addin](https://github.com/papauorg/Cake.Liquibase){:target="_blank"}. It does not wrap all the functionality liquibase provides, but the part I needed, updating the database, is included. It allows defining the liquibase parameters by setting properties on a settings class and calling a method instead of having to deal with a batch file, the proper working directory, environment variables, escaping parmeters or quotes around paths. It's just more comfortable to use the addin. And this is true for the most addins or tools.

Of course the addins and tools provided via NuGet are changed every now and then. To keep your build files stable you can use fixed versions of them by defining it in your build files.

```csharp
#addin "nuget:?package=Cake.AddinName&version=1.0.0"
```

This avoids broken builds because of incompatible updates of an addin. The same can be done for the Cake version itself. But there you have to specify the version in the `packages.config` within the `tools` folder with the usual NuGet xml sytax.
```xml
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <packages>
      <package id="Cake" version="0.21.1" />
  </packages>
</packages>
```
Additionally, you can include other dependencies here. For example: I included the Liquibase.Cli package that contains the liquibase binaries here, so the Cake.Liquibase addin can use them and they do not need to be installed on the machine (also the addin supports installed versions, too). If you change the `packages.config` file keep in mind that you add it to your source control, but leave out the rest of the tools folder as those will get downloaded by the bootstrapper and do not need to be checked in. Also see the [Pinning Cake version](https://cakebuild.net/docs/tutorials/pinning-cake-version){:target="_blank"} topic in the documentation.

# Centralized build logic
In the past we had build files, too, but they didn't contain the whole build process. They contained e.g. building the solution, but running tests or trying to run the database scripts were steps in the build server. This is working fine, but in my opinion has some disadvantages.

The first one is, if a step is only available on your build server and it fails, it is much harder to reproduce and fix it on a developer machine. If you keep all steps in the build file you can also execute them the same way they happen on the build machine. This can help a lot and speed up the time needed to fix build issues in many cases.
{% include image.html name="centralize_build_steps.png" caption="Centralize build steps - before and after" %}
Another disadvantage is that you have to change the build on the server in sync with the version of your source and build file. Why is that? Example incoming! 

You want to switch your test framework because you like the shiny new release of **SuperTest** so much better than the old **UsualTest** that you currently use. Well, you go ahead and change all tests to use **SuperTest** and remove all references to **UsualTest**. You install the **SuperTest** runner on your machine, give it a try and commit because everything worked like a charm. Great right? No! Now, the pull request validation build fails because it's still using that old school **UsualTest** runner. Damn. Good thing it's only the pull request validation build, so just change the runner and be done with it! A few minutes later the colleague from the next room storms in and gives you a karate chop in the neck because his pull request was rejected due to failing tests. Similar problems occur for the continuous-, daily-, release- and other builds but only after your change is merged to the different branches. You see where I'm going with this.
You could create builds with different runners that build different branches depending on different build variables or something like that, but that makes the build environment very confusing. Also someone has to maintain those builds. 

A better alternative, in my opinion, would be to keep this logic in the build file. So the build server just executes the script. The developer chooses the dependencies, tools and steps to be executed by editing the script and checking it in alongside with the code changes. All the build logic in one place and executable from the server or developer machine alike. This way the guy next door can still finish his pull request because his build file still uses the **UsualTest** runner. You can use the **SuperTest**s and its runner in your file. And when your branches get merged, the old tests are updated with your changes and the new runner gets introduced to the build file and you and your colleague live happily ever after. And so do the other builds.

I also sneaked in another goodie in the above example, you can have your build logic under source control with the obvious advantages. Big plus here!

# Bootstrap
To use all the above mentioned advantages the developers must be able to use these scripts on their machines as well. This is very simple with Cake. It provides a powershell or bash script that handles all the bootstrapping. All a developer must do is execute that script. No installation of Cake is required. The script downloads the latest or in the packages.config specified version of Cake, your addins and tools and calls Cake afterwards. It's really easy to get started.

# Debugging
As your build files get more complex you may need to find errors in it from time to time. Good news here: You can even debug your build scripts with Visual Studio. For this to work open the Cake script in Visual Studio. Pro-Tip: Install the Cake for Visual Studio extension first! Then start the Cake script with the `--debug` switch. Set a breakpoint and attach the debugger to the process Cake tells you. 

{% include image.html name="Debug_Cake.gif" caption="Debugging Cake scripts" %}

This is really helpful and a fun thing to do! Try it! 

# More than build
Other than just handling builds we also use the script for other tasks. For example we regularly need to setup or recreate databases for our development environments. Instead of creating separate scripts and tools for that, we use Cake now. That's why I created the Cake.Liquibase addin for. It's just another target in the Cake script. 

```powershell
.\build.ps1 --Target UpdateDatabase
```
That's all there is to be done for it. You could automate all kinds of tasks to setup your development environment for a specific project. If it's more complex you can also use arguments to specify details. 

```powershell
.\build.ps1 --Target UpdateDatabase --databaseName=TestDatabase --otherParameters=y
```
The best thing is you can also document such tasks directly where they are used. Tasks in Cake can be documented with the `Description()` extension method and the `--showdescription` parameter displays all tasks with their descriptions. 

```csharp
Task("UpdateDatabase")
  .Description("Creates a new local development database ...")
  .Does(() => {
    // code here
  });
```
gives you

```powershell
> .\build.ps1 --showdescription
Task                          Description
================================================================================
UpdateDatabase                Creates a new development database when necessary
                              and runs the database update scripts against it. 
                              This gives you a clean and up to date database 
                              to develop against.
Default                       
```

Unfortunately, there is not a built in way to [document arguments](https://github.com/cake-build/cake/issues/1708){:target="_blank"} the same way as tasks at the time of writing. But because it's .NET you can just use the available tools [like described here](https://stackoverflow.com/questions/45413524/how-can-cake-build-arguments-be-documented){:target="_blank"} for that. That goes for any task you want to put in your script. Because it's .NET the possibilities are nearly endless :P.

# Conclusion
Cake keeps our build servers clean, the dependencies are managed via NuGet, we can write C# instead of XML, we can write extensions so it fits our needs, we have an all-round tool for managing our projects and it is documented properly within the script. Any questions?