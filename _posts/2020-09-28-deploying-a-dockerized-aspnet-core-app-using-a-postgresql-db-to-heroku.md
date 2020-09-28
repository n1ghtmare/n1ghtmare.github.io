---
layout: post
title: "Deploying a Dockerized ASP.NET Core app using a PostgreSQL DB to Heroku"
description: "Setup and deployment of an ASP.NET Core app that is setup as a Docker container and uses PostgreSQL as a DB to Heroku from a git branch on Github"
date: 2020-09-28
tags: [programming, docker, asp.net, .net-core, postgresql, heroku]
comments: false
share: true
---

I'm working on an ASP.NET Core application, that has PostgreSQL as a database. My application is setup to run in a Docker container. I was looking for a way to deploy the app to Heroku (I'm a big fan!). I searched the internet and found a bunch of articles/blog posts/tutorials on the subject, but all of what I found had no mention of connecting to a DB. It took me a little while to set everything up and make it work.

Here I'm going to try to document how I did the deployment and how I've setup the database.

First I created an app on Heroku and then added Heroku-Postgres as a resource to it. If you go to the new datasource and check its settings you will notice that in the section about DB credentials it says that the credentials are "not permanent" and that "Heroku rotates credentials periodically and updates applications where this database is attached". This confused me at first, since I thought it will be as simple as just specifiying a production DB connection string in my web app and forget about it. However, that's not how you should approach it on Heroku. When you deploy a Heroku container (dyno) it comes with a bunch of environment variables setup that (as stated earlier) will change periodically. You can see those environment variables under "Settings" -> "Config Vars", and if you take a look you will see that Heroku has created a `DATABASE_URL` variable that contains the connection string for the database. We need to use *this* connection string in our ASP.NET application when it's deployed (instead of reading it from our `appsettings.json` file). In addition to those configuration variables, there are further [Heroku Platform configuration variables](https://devcenter.heroku.com/articles/platform-api-reference#config-vars) (such as for example which port your container is running on or your App id etc.).

In addition, I wanted to deploy the application automatically from a specific git branch on Github (where I have the source code for the application). Heroku is really cool and has the option to integrate with Github. You can find that option under "Deploy" -> "Deployment Method" and select Github, then follow the steps to finish the connection.

Next, I downloaded the Heroku CLI and set my application stack to be a `container`, by running: 

```
heroku stack:set container
```

Finally, I've created a `heroku.yml` config file that just has the following:

```yaml
build:
    docker:
        web: Dockerfile
```

This is so that Heroku knows how to build the application when deploying (more details [here](https://devcenter.heroku.com/articles/build-docker-images-heroku-yml)).

Now, that I had the Heroku setup done and I moved on to setting up my application.

The first thing I did, is change how my database connection string is retrieved on production (I have this `ConfigurationService` that I use across my application):

```cs
public class ConfigurationService : IConfigurationService {
    private readonly IConfiguration _configuration;
    private readonly IWebHostEnvironment _webHostEnvironment;
    
    public ConfigurationService(IConfiguration configuration, IWebHostEnvironment webHostEnvironment) {
        _configuration = configuration;
        _webHostEnvironment = webHostEnvironment;
    }

    private string GetHerokuConnectionString() {
        // Get the connection string from the ENV variables
        string connectionUrl = Environment.GetEnvironmentVariable("DATABASE_URL");

        // parse the connection string
        var databaseUri = new Uri(connectionUrl);

        string db = databaseUri.LocalPath.TrimStart('/');
        string[] userInfo = databaseUri.UserInfo.Split(':', StringSplitOptions.RemoveEmptyEntries);

        return $"User ID={userInfo[0]};Password={userInfo[1]};Host={databaseUri.Host};Port={databaseUri.Port};Database={db};Pooling=true;SSL Mode=Require;Trust Server Certificate=True;";
    }

    public string DatabaseConnectionString =>
        _webHostEnvironment.IsDevelopment()
            ? _configuration.GetConnectionString("DefaultConnection")
            : GetHerokuConnectionString();
    
    // other configuration settings...
}
```

Now, my application will use the connection string from the Environment variable when deployed to production.

At this point, I thought everything is good, deployed, tested and became dissapointed by the big fat error on production. Turns out that Heroku assigns a port for your container to run on dynamically (remember the Heroku Platform variables mentioned earlier).

I proceeded by changing my web host builder to read the `$PORT` environment variable on production, as follows:

```cs
public class Program {
    public static void Main(string[] args) {
        CreateWebHostBuilder(args).Build().Run();
    }

    private static bool IsDevelopment => 
        Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") == "Development";

    public static string HostPort => 
        IsDevelopment 
            ? "5000" 
            : Environment.GetEnvironmentVariable("PORT");

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseUrls($"http://+:{HostPort}")
            .UseSerilog((context, config) => {
                config.ReadFrom.Configuration(context.Configuration);
            })
            .UseStartup<Startup>();
}
```

This is how my `Dockerfile` looks like:

```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build
WORKDIR /app

WORKDIR /src
COPY ["MyProject.WebUI/MyProject.WebUI.csproj", "MyProject.WebUI/"]
COPY ["MyProject.Core/MyProject.Core.csproj", "MyProject.Core/"]
COPY ["MyProject.Database/MyProject.Database.csproj", "MyProject.Database/"]
RUN dotnet restore "MyProject.WebUI/MyProject.WebUI.csproj"
COPY . .
WORKDIR /src/MyProject.WebUI

# Setup Node (13.x)
# see https://github.com/nodesource/distributions/blob/master/README.md#deb
RUN apt-get update -yq 
RUN apt-get install curl gnupg -yq 
RUN curl -sL https://deb.nodesource.com/setup_14.x | bash -
RUN apt-get install -y nodejs

RUN npm install -g npm webpack webpack-cli
RUN npm install
RUN npm run-script build
RUN dotnet build "MyProject.WebUI.csproj" -c Release -o /app

FROM build AS publish
RUN dotnet publish "MyProject.WebUI.csproj" -c Release -o /app

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1 AS runtime
WORKDIR /app
COPY --from=publish /app .

ENTRYPOINT ["dotnet", "MyProject.WebUI.dll"]
```

I have setup for Node, since I'm having a React front end with this app, and I have multiple assemblies referenced. The `Dockerfile` is not really relevant to the the build and to Heroku, as long as it builds, you're good to go.

The last thing I did was to setup [Serilog](https://serilog.net/) (as you can see in the `Program.cs`) to only log into console (you can see those logs in Heroku), rather than a file.

At this point, I deployed and everything worked as expected.
