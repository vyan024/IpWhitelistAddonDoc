# IPWhitelist 

This **free** module is designed for blocking/restricting access on a per site basis for `Optimizely CMS 12+` solution hosted on Optimizely DXP with the following features

* Whitelist single IP address per site basis
* Whitelist a range of IP addresses via CIDR per site basis
* Whitelist known paths (e.g. /episerver/health)


[<img src="https://i.imgur.com/nnY2vzF.png" width="600"/>](https://i.imgur.com/nnY2vzF.png)

- [IPWhitelist](#ipwhitelist)
  - [How to start](#how-to-start)
  - [How to configure](#how-to-configure)
    - [Option 1 - Block site access in appsettings.json](#option-1---block-site-access-in-appsettingsjson)
    - [Option 2 - Block site access in Admin UI](#option-2---block-site-access-in-admin-ui)
    - [Option 3 - Configure programmatically](#option-3---configure-programmatically)
  - [How it works](#how-it-works)
  - [Configuration](#configuration)
    - [Startup.cs](#startupcs)
    - [Module access control](#module-access-control)
  - [Troubleshooting](#troubleshooting)
- [Support](#support)
- [Limitations](#limitations)


[![Microsoft:.NET](https://img.shields.io/badge/.NET%206-grey?style=for-the-badge&label=.NET%20Platform&labelColor=grey&color=5e31f6)](https://dotnet.microsoft.com/en-us/)    [![Optimizely](https://img.shields.io/badge/12-grey?style=for-the-badge&label=Optimizely%20CMS&labelColor=grey&color=0037fe)](https://www.optimizely.com/)  [![IPWhitelist: Nuget](https://img.shields.io/nuget/v/IPWhitelist.svg?style=for-the-badge)](https://www.nuget.org/packages/IPWhitelist/) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)



## How to start
Install the add-on nuget package into your **Optimizely Cms Web** project, then make two lines of code change in your `Startup` file (see details in <a href="#configuration">`Configuration`</a> section). Finally, rebuild and re-run your site should be working as normal without noticing any change. 

## How to configure 
Once the package has been installed and configured, you have a few options to turn on the IP restriction. 

> **Caution**: 
> 
> By default, all sites are **accessible** after installing the module, you will need to explicitly turn off access for each site via the configuration settings or Admin UI.

### Option 1 - Block site access in appsettings.json

> **Tips**
> 
> `Appsetting.json config option will always take precedence than other options.`

| Name                     	| Default value     	| Description                                                                                                             	|
|--------------------------	|-------------------	|-------------------------------------------------------------------------------------------------------------------------	|
| AllowByPassLocalRequest  	|        true       	| Get or set the flag which decides whether local request should be allowed to access. `true` by default.                 	|
| WhitelistSiteDefinitions 	|                   	| Get or set a list of `WhitelistSiteDefinition` that are used to decide whether a specific site should allow to access.  	|
| AdminSafeIpList          	|                   	| Get or set a list of Ips (e.g. Company's VPN IP, static IP) that allows to bypass the IpWhitelist module check.         	|
| IgnorePaths              	| /episerver/health 	| Get or set a list of paths that allows to ingore the check by IpWhitelist module. `["/episerver/health"]` by default.   	|

**Example of configuration**
```json
"WhitelistSiteOptions": {
  "AllowByPassLocalRequest": false,
  "WhitelistSiteDefinitions": [
    {
      "SiteId": "e68c3e92-c76c-44c0-9196-338d41611e1c",
      "AllowAccess": true
    }
  ],
  "AdminSafeIpList": ["10.1.1.1"],
  "IgnorePaths": ["/episerver/health"]
}
```

### Option 2 - Block site access in Admin UI

You can control a specific site access from General menu (see screenshot below)

[<img src="https://i.imgur.com/xKYlew0.png" width="600"/>](https://i.imgur.com/xKYlew0.png)

### Option 3 - Block site access programmatically
You can configure and overrides the value specified in appsettings programmatically by supplying `WhitelistSiteOptions`

```csharp
services.AddIpWhitelist(options =>
{
    options.WhitelistSiteDefinitions = new List<WhitelistSiteDefinition>
    {
        new() { SiteId = "e68c3e92-c76c-44c0-9196-338d41611e1c", AllowAccess = false }
    };
    options.AllowByPassLocalRequest = true;
});

```

## How it works

There are 5 default evaluators that implement `IRequestEvaluator` interface. All requests will be processed sequentially through each evaluator based on their priority level, starting from the highest to lowest priority.

When a request is coming, the middleware will pass it onto the `DefaultRequestEvaluator` engine which determines the request authorization through each of the evaluator. It will let the request come through as soon as the first evaluator returns `true` 

| Name                       | Priority | Description                                                                                                                                                    |
|----------------------------|-------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| LocalRequestEvaluator      | 2000  | This evaluator is utilized to verify whether the current request coming from local request. It is primarily design to simplify the local development by skipping the restriction. It's the `first` evaluator gets executed by the engine. You can disable this by setting `AllowByPassLocalRequest` to `false` in `appsettings.json` |
| EnvironmentByPassEvaluator | 1000  | This evaluator is utilized to verify whether the current request aligns with the value specified in the `appsettings.json - WhitelistSiteDefinitions` and the `Admin UI - General`, thereby determining its authorization to access a specific site.                                                                 |
| IgnorePathEvaluator        | 900   | This evaluator is utilized to verify whether the current request's resource aligns with the value specified in the `appsettings.json - IgnorePaths` and the `Admin UI - Ignore Path(s)`, thereby determining its authorization to access a specific site.                                  |
| IpAddressEvaluator         | 200   | This evaluator is utilized to verify whether the current request's IP address matches with the value specified in the `appsettings.json - AdminSafeIpList`  and the `Admin UI - IP(s)`, thereby determining its authorization to access a specific site.                                          |
| CidrAddressEvaluator       | 100   | This evaluator is utilized to verify whether the current request's IP address matches the value specified in the `Admin UI - CIDR(s)`, thereby determining its authorization to access a specific site.                          |


## Configuration

### Startup.cs
After installing the package `IPWhitelist` in your project, you need to ensure the following lines are added to the startup class of your solution:

```csharp
public void ConfigureServices(IServiceCollection services)
{
   // .... other services
    
   services.AddIpWhitelist();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Add IPWhitelist middleware before any other middleware
    app.UseIpWhitelist();
    // ... other middleware
}

```
The call to ```services.AddIpWhitelist()``` sets up the dependency injection required by the `IPWhitelist` to ensure the solution works as intended.  This works by following the Services Extensions pattern defined by Microsoft.


### Module access control

By default, only users in `CmsAdmins`, `WebAdmins` and `Administrators` role can access Admin UI. 
If you would like to use you custom policy, you can customize it with `AuthorizationPolicyBuilder` when registering the `IPWhitelist`.

```csharp
services.AddIpWhitelist(builder =>
{
    builder.RequireRole("mycustomrole");
    //builder.AddRequirements()
});
```
## Troubleshooting
The module has extensive logging. Turn on information logging for the `IPWhitelist` namespace in your logging configuration.

```json
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "IPWhitelist":"Information" // Add this line
    }
  }
```
# Support

 IPWhitelist is a **free** module. If you like this module and keen to support me, you can get me a coffee :)

 <a href="https://www.buymeacoffee.com/javafun"><img src="https://img.buymeacoffee.com/button-api/?text=Buy me a coffee&emoji=☕&slug=javafun&button_colour=FF5F5F&font_colour=ffffff&font_family=Comic&outline_colour=000000&coffee_colour=FFDD00" /></a>

# Limitations

* This module is **NOT** designed to work with `Optimizely CMS 11` or below.

