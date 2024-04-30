---
title: Setting up basic logging with Serilog
draft: false
tags: 
  - C-Sharp
  - Logging
  - Serilog
--- 
## Adding richer logging to dotnet apps with Serilog
### Setup
First start off with installing the nuget packages relevant to your solution
- `dotnet add package Serilog` is the base packages and required for this.
- `dotnet add package Serilog.AspNetCore` if you're using dotnet core.
- `dotnet add package Serilog.Extensions.Hosting` if you want to add logging via the Host dependency container.
- `dotnet add package Serilog.Settings.Configuration` if you want to pull Serilog configuration from an injected `appsettings.json´ file.

### Sinks
Serilog uses systems called sinks to output the logs. For example, the `Serilog.Sinks.Console` package provides methods and extensions for outputting to the console, and the `Serilog.Sinks.File` provides methods and extensions for outputting to files. There are more sinks, but these two are the basic ones used.

### Configuring the logger
The logger can be configured two ways - programmatically using fluent functions or from the `appsettings.json`.

#### `appsettings.json` configuration
By adding a `Serilog` entry to your `appsettings.json` file, you can easily configure and share the logger's behaviour.
```json
"Serilog": {  
  "MinimumLevel": "Debug",  
  "Using": ["Serilog.Sinks.Console"],  
  "WriteTo": [  
    {      
      "Name": "Console",  
      "Args": {  
        "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}"  
      }  
    }  
  ],  
  "Enrich": ["FromLogContext", "WithMachineName", "WithProcessId"]  
}
```
Here is a basic setup for logging via console. The `WriteTo.[n].Args.OutputTemplate` defines how your lines of logging will be presented. There are many different tokens and keyworks that the templating can use to customize the output of the console logging.
### Registering the logger
#### WebApplication
In your service registration, you want to add the logger to the app host. The host object is available on the Application builder you're using, for example in this case the `WebApplicationBuilder`.
```c# title="Program.cs"
builder.Host.UseSerilog((context, _, configuration) =>  
{  
    configuration        
	    .ReadFrom.Configuration(context.Configuration);  
});
```
This will then register Serilog to use the settings provided in the `appsettings.json` that is injected to the configuration. 

#### IHostApplication
For Host applications you configure the `Logger` property directly on the builder. 
```c# title="Program.cs"
builder.Logging.ClearProviders();  
  
var loggingConfig = new LoggerConfiguration()  
    .ReadFrom.Configuration(builder.Configuration)  
    .CreateLogger();  
  
builder.Logging.AddSerilog(loggingConfig);
```
First the default logging behaviour is cleared, a `LoggerConfiguration` class is instantiated and the `appsettings.json` that is injected into the configuration is used. Finally the Serilog logger is created by providing the config. 

### Usage via dependency injection
Once the logger is configured and registered, it can be injected into other services via construction injection.
```c# title="MyController.cs"
public class MyController : Controller
{
	private readonly ILogger<MyController> _logger;
	
	public MyController(ILogger<MyController> logger)
	{
	    _logger = logger;
	}

	[HttpGet]
	public async Task<IActionResult> Get()
	{
	     _logger.LogInformation("MyController Endpoint hit!");
		
	     return Ok("Hello world");
	}
}
```