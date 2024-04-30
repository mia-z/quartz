---
title: Using SCSS in a Blazor app
draft: false
tags: 
  - CSharp
  - SCSS
  - dotnet
--- 
## Setting up
First install the package for transpiling the scss files. 

`dotnet add package AspNetCore.SassCompiler`

Note that this will also add the dart runtime binaries as this uses dart-sass under the hood.
### Initialization
Since this is only used for transpiling the scss/sass files, you only want to run it in development environments, so register it with a debug `#if` directive during service registration.

```c#
//Code omitted
#if DEBUG  
	builder.Services.AddSassCompiler();  
#endif
```

### Configuration
The settings for the transpiler live in your `appsettings.json` file - ideally `appsettings.Development.json`. A basic example could look like this

```json {7,11,12} title="appsettings.Development.json"
{
	"SassCompiler": {  
	  "Source": "Styles",  
	  "Target": "wwwroot/styles",  
	  "Arguments": "--style=compressed",  
	  "GenerateScopedCss": true,  
	  "ScopedCssFolders": ["Pages", "Components", "Layout", "../MySolution.MyOtherClassLibraryProject/Pages"],  
	  "IncludePaths": [],  
	  "Compilations": [
	    { 
          "Source":  "wwwroot/styles/main.scss", 
          "Target":  "wwwroot/styles/main.css" 
        }  
	  ],
	  "Configurations": {  
	    "Debug": {   
          "Arguments": "--style=expanded"  
        }
	  }
	}
}
```

There are two main compilation areas declared here
- The `Compilations` array defines an input file and an output file, denoted by `Source` and `Target`, respectively. These are going to want to be referenced in your html somewhere (make sure to reference the compiled .css file, and not the .scss file!)
```razor title="App.razor"
<!-- code omitted -->
<link rel="stylesheet" href="styles/main.css" />
```

- The `ScopedCssFolders` array defines a collection of folders where scoped css can be found and transpiled. 
  If you're using scoped css in another project, you can also reference relative paths here.
  With this in place you can now just write `Index.razor.scss` and the transpiler will generate `Index.razor.css`, as well as a source map for the sass file. 
  Make sure you also reference scoped css in your main razor file
```razor title="App.razor"
<!-- code omitted -->
<link rel="stylesheet" href="MySolution.MyProject.styles.css" />
```

With all this in place you should now be able to write sass instead of css and it all work seamlessly!