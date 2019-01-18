# Implementing Composition in your Web Project the right way
I have been using `System.ComponentModel.Composition` (previously MEF) for quite some time now, and it really
is a very useful library that greatly adds flexibility to your system if used the right way.

In this article, I assume you have some knowledge of the `System.ComponentModel.Composition`
library and are here because you are now trying to use it in your ASP.NET Core, ASP.NET MVC, Web API
or classic ASP.NET project and want to be sure you are doing it right.

## The challenge
I have struggled more than once to find a good example of how to integrate MEF into an application,
especially a web application. Now of course it could be that I just didn't enter the right search
phrase in Google and never found the right articles. But the examples I found often suggested 
solutions I consider anti patterns when implementing DI of composition, for example passing the 
composition container through to objects so that they themselves can ask the container to 
compose them.

A correct implementation of Composition, in my opinion, adhers to a few fundamental design principles:
- Composing objects is the sole responsibility of one single class in an application. In other words, you will find the code `container.ComposeParts(someObject)` only once in the application's source code.
- No other class in the application has any knowledge whatsoever of the composition container. 
- If at all possible, the composing of objects should be handled by middleware. In an ASP.NET Core application, that means integrating composition into the request lifecycle.

Below I will discuss some ways to integrate composition into a web application, adhering to above principles,
starting with the most modern technology, ASP.NET Core MVC / Web Api, then ASP.NET MVC and Web Api
and even a classic ASP.NET application using web pages. 

But first, let's present a simple use case

## Use case
<img align="right" style="margin-left: 5px;" src="https://upload.wikimedia.org/wikipedia/commons/6/6a/Whiskey_bottling_plant.jpg">Let's
imagine we have to develop an application that monitors a bottling installation at a factory
that bottles our favourite beverage. Of course, every bottle must be filled with the correct
amount of liquid. So the people at the factory have installed a camera that can register how full
every bottle is. And now they want a big screen in their office where they can see what the
camera is measuring.

Now we know that soon enough they will also be adding some scales, to weigh every bottle. And
after that, they will even be installing a laser and optical sensors to measure the quality of 
the liquid in the bottle.

We can't support all these devices up front, so we decide we want some flexibility, and 
want to be able to release new libraries supporting new devices at some time, without replacing 
the entire application.

What we do know is that all these devices roughly have the same functionality and 
operating them allways follows a similar pattern. So we have decided to write proxy classes for
them that all adhere to a standard interface. Extremely simplified, it looks like this:

```csharp
public interface IDeviceProxy
{
  string DeviceType { get; }
  bool Reset(int deviceId);
  int Read(int deviceId);
}
```

### Camera support
The first device we will be supporting, is the camera. Our camera class is as follows:

```csharp
using System.ComponentModel.Composition;

namespace MyApp
{
  [Export(typeof(IDeviceProxy))]
  public class CameraProxy : IDeviceProxy
  {
    public string DeviceType => "camera";

    public int Read(int deviceId)
    {
      return 701;
    }

    public bool Reset(int deviceId)
    {
      return true;
    }
  }
}
```

This is of course overly simplified. Our camera always returns 701 (ml?) as measurement, and
a reset always succeeds. Life is sweet. 

Like I said, I assume you know how .NET's `System.ComponentModel.Composition` works, so
the addition of the `[Export]` attribute to our `CameraProxy` class should not surprise you. They
ensure later on our `CameraProxy` will be added to the catalog by `System.ComponentModel.Composition`.

Now, let's look at how we would consume this class in variour variations of ASP.NET using composition.

## ASP.NET Core MVC / Web Api
.NET Core comes with it's own integrated Dependency Injection framework. You may wonder why you need 
`System.ComponentModel.Composition` in the first place. Well, for some simple cases, you don't. But 
when you want more flexibility, for instance in when, how and where dependency will be deployed, 
`System.ComponentModel.Composition` has some unique advantages. And, as you will see, we will make use 
of .NET Core's own DI framework to get `System.ComponentModel.Composition` neetly injected into the 
request pipeline. 

ASP.NET MVC has a lot of extension points that allow us to control the behavior of the framework, for
example how Controllers are instantiated. We can control the creation of controllers by writing our
own class that implements IControllerFactory and adding that to the request pipeline using .NET Core's
own DI system.

Our custom Controller factory that handles the composition of our controllers, is as follows:

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel.Composition;
using System.ComponentModel.Composition.Hosting;
using System.IO;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Controllers;
using Microsoft.AspNetCore.Mvc.Internal;

namespace MyApp
{
  public class CompositionControllerFactory : DefaultControllerFactory
  {
    CompositionContainer _container;

    public CompositionControllerFactory(IControllerActivator controllerActivator, IEnumerable<IControllerPropertyActivator> propertyActivators) 
      : base(controllerActivator, propertyActivators)
    {
      this._container = ConfigureContainer();
    }

    public override object CreateController(ControllerContext context)
    {
      var controller = base.CreateController(context);

      _container.ComposeParts(controller);

      return controller;
    }

    private static CompositionContainer ConfigureContainer()
    {
      var assembly = System.Reflection.Assembly.GetExecutingAssembly();
      var uri = new Uri(assembly.GetName().CodeBase);

      var catalog = new AggregateCatalog(
        new AssemblyCatalog(assembly),
        new DirectoryCatalog(Path.GetDirectoryName(uri.LocalPath), "MyApp.Devices.*.dll")
      );

      return new CompositionContainer(catalog, true);
    }
  }
}
```

Our `CompositionControllerFactory` derives from `DefaultControllerFactory`, because the
default way the framework creates Controllers is fine for us, we just want to do one little extra 
thing after the controller is created, that being satisfying the imports the controller requires.

Now we need to insert our custom controller factory into the request pipeline.

If you have a standard ASP.NET Core MVC / Web Api web application, for instance one that you 
created by selecting "New ASP.NET Core Web Application" and then the "Web Application (Model-View-Controller)"
template in Visual Studio, you will have a `Startup` class similar to the one below. Notice the
added line however.

```csharp
public class Startup
{
  public Startup(IConfiguration configuration)
  {
    Configuration = configuration;
  }

  public IConfiguration Configuration { get; }

  // This method gets called by the runtime. Use this method to add services to the container.
  public void ConfigureServices(IServiceCollection services)
  {
    /// ...
    services.AddSingleton<Microsoft.AspNetCore.Mvc.Controllers.IControllerFactory, CompositionControllerFactory>();
    services.AddMvc();
  }

  // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
  public void Configure(IApplicationBuilder app, IHostingEnvironment env)
  {
    /// ...
  }
}
```

I have removed the contents of the Configure methods for brevity. 

The extra line right before the call to `services.AddMvc()` adds our own ControllerFactory
as a service. When the call to `AddMvc()` is made, ASP.NET MVC will not add it's default ControllerFactory, 
but use the one that is already available.

Next, create a controller to test it works:

```csharp
using System.Collections.Generic;
using System.ComponentModel.Composition;
using System.Linq;
using Microsoft.AspNetCore.Mvc;

namespace MyApp.Controllers
{
  [Route("api/[controller]")]
  [ApiController]
  public class DeviceController : ControllerBase
  {
#pragma warning disable 649
    [ImportMany]
    private IEnumerable<IDeviceProxy> _deviceProxies;
#pragma warning restore 649

    [HttpGet("{deviceType}/{deviceId}")]
    public IActionResult Read(string deviceType, int deviceId)
    {
      var proxy = _deviceProxies.Where(p => p.DeviceType == deviceType.ToLowerInvariant()).FirstOrDefault();
      if (proxy == null)
      {
        return BadRequest();
      }

      var result = proxy.Read(deviceId);

      return Ok(result);
    }

    [HttpGet("{deviceType}/{deviceId}/reset")]
    public IActionResult Reset(string deviceType, int deviceId)
    {
      var proxy = _deviceProxies.Where(p => p.DeviceType == deviceType).FirstOrDefault();
      if (proxy == null)
      {
        return BadRequest();
      }

      var result = proxy.Reset(deviceId);

      return Ok(result);
    }
  }
}
```

Notice the `#pragma` directive around the field flagged with the `[ImportMany]` attribute. 
Unfortunately, the compiler is not that smart that it recognizes `System.ComponentModel.Composition` imports,
and will generate a CS0649 warning when compiling this code, saying

```
CS0649	Field 'DeviceController._deviceProxies' is never assigned to, and will always have its default value null
```

That's anoying, and untrue, because we know our `CompositionControllerFactory` will be used to satisfy the import
and the code will run. So we have added the `#pragma` to turn the warning of.

Run the app, set a breakpoint if you wish, and navigate to the `api/device/camera/1` page to see it in operation.
