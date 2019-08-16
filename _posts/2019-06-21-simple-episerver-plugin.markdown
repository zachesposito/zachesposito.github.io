---
layout: post
title:  "Creating a simple EPiServer plugin"
date:   2019-06-21
subtitle: Automating data flow to AWS using S3 and Lambda
---

One of my main projects is a site that was built on the EPiServer CMS a couple years ago. The site depends on a set of product data, and I wanted to give site admins the ability to manage that data in the EPiServer admin interface. I knew EPiServer could be extended with plugins, but I had a difficult time figuring out the exact steps to make my own plugin. 

Eventually I discovered that making a basic plugin is actually really simple. It only requires a controller and a view, plus two more key pieces: authorization on the controller, and a menu provider.

Here is a simplified version of the steps I took to create my plugin:

## Make the controller
{% highlight c# %}
using System;
using System.Web;
using System.Web.Mvc;

namespace MyEPiServerSite.Controllers
{
	[Authorize(Roles = "Administrators")]
	public class ProductDataAdminController : Controller
	{
		public ActionResult Index()
		{
			return View();
		}
	}
}
{% endhighlight %}

A regular .NET MVC controller. I decorated it with the `Authorize` attribute to limit access to CMS admins only.

Don't forget to set up its route, for example in `global.asax`:
{% highlight c# %}
protected override void RegisterRoutes(RouteCollection routes)
{
  /*...*/
  routes.MapRoute(
    "ProductData",
    "product-data/{action}",
    new { controller = "ProductDataAdmin", action = "Index" }
  );
}
{% endhighlight %}

## Make the view
View itself:
{% highlight html %}
@{
  ViewBag.Title = "Product Data Admin";
  Layout = "~/Views/ProductDataAdmin/_layout.cshtml";
}
<h2>Index</h2>
Hello this is the product data admin plugin!
{% endhighlight %}

_layout.cshtml (uses EPiServer's header):
{% highlight html %}
@using EPiServer.Framework.Web.Resources
@using EPiServer.Shell.Navigation
<!DOCTYPE html>
<html>
<head>
  <meta name="viewport" content="width=device-width" />
  <title>@ViewBag.Title</title>
  <!-- Shell -->
  @Html.Raw(ClientResources.RenderResources("ShellCore"))
  @Html.Raw(ClientResources.RenderResources("ShellWidgets"))
  <!-- LightTheme -->
  @Html.Raw(ClientResources.RenderResources("ShellCoreLightTheme"))
  <!-- Navigation -->
  @Html.Raw(ClientResources.RenderResources("Navigation"))
  <!-- Dojo Dashboard -->
  @Html.Raw(ClientResources.RenderResources("DojoDashboardCompatibility", new[] { ClientResourceType.Style }))
</head>
<body>
  @Html.Raw(Html.ShellInitializationScript())
  @Html.Raw(Html.GlobalMenu())
  <div>
    @RenderBody()
  </div>
</body>
</html>
{% endhighlight %}

## Make the menu provider
This adds a link to the plugin in the top nav bar in EPiServer:
ProductDataMenuProvider.cs:
{% highlight c# %}
using EPiServer.Security;
using EPiServer.Shell.Navigation;
using System;
namespace TRUEpiserver.Business.Providers
{
	[MenuProvider]
	public class ProductDataMenuProvider : IMenuProvider
	{
		public IEnumerable<MenuItem> GetMenuItems()
		{
			var toolbox = new SectionMenuItem("Product Data", "/global/product-data")
			{
				IsAvailable = (request) => PrincipalInfo.HasEditorAccess
			};
			var home = new UrlMenuItem("Home", "/global/product-data/home", "/product-data/home")
			{
				IsAvailable = (request) => PrincipalInfo.HasEditorAccess
			};
			return new MenuItem[] { toolbox, home };
		}
	}
}
{% endhighlight %}

That's all it takes to create a super-basic EPiServer plugin!