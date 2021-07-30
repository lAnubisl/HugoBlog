---
title: "3 steps to create integration tests for your ASP.NET MVC 5 application"
date: 2016-10-02T09:43:29+03:00
draft: false
Summary: "![](/images/3-steps-integration-testing-aspnet-mvc/integration-testing-mock-component.png)

When you develop an ASP.NET MVC application you should test it anyway. You can cover different parts of your application logic with unit tests or you can create tests that look like user interaction scenarios. These tests have several advantages over unit tests:

* These tests are independent of implementation and you can't break your test by refactoring (the only scenario when you should modify your test is functional requirement change).

* These tests are good documentation for your application.

* These tests can give you idea about that your users need and how will they supposed to use concrete feature."
---

![](/images/3-steps-integration-testing-aspnet-mvc/Integration-testing-1.jpg)

When you develop an ASP.NET MVC application you should test it anyway. You can cover different parts of your application logic with unit tests or you can create tests that look like user interaction scenarios. These tests have several advantages over unit tests:

*   These tests are independent of implementation and you can't break your test by refactoring (the only scenario when you should modify your test is functional requirement change).
*   These tests are good documentation for your application.
*   These tests can give you idea about what your users need and how will they supposed to use the specific feature.

## Step 1\. Host your application for testing

The easiest way to host ASP.NET MVC application is to use IIS Express. To do this you need to run:

```
iisexpress.exe path:"{0}" port:{1}
```

where **path** is the physical path to your application sources and **port** is the port number you would like IIS Express to use. 

In a test project you can run this as follows:

``` csharp
 var programFiles = Environment.GetFolderPath(Environment.SpecialFolder.ProgramFiles);
 var iisProcess = new Process();
 iisProcess.StartInfo.FileName = programFiles + @"\IIS Express\iisexpress.exe";
 iisProcess.StartInfo.Arguments = $"/path:\"{applicationPath}\" /port:{8080}";
 iisProcess.Start();
```

You can use the following to calculate absolute physical path from relative path to your web application:

``` csharp
string GetApplicationPath()
 {
     return Path.GetFullPath(
         Path.Combine(
             GetCurrentDirectory(), 
             "../../../MyApplication.Web"
         )
     );
 }

 string GetCurrentDirectory()
 {
     return Path.GetDirectoryName(
         Uri.UnescapeDataString(
             new UriBuilder(
                 Assembly.GetExecutingAssembly().CodeBase
             ).Path
         )
     );
 }
 ```

there is one little thing we should do to be sure that application is fully initialized. We should make the first request to it. Why this is important will be shown later in step 3.

``` csharp
 using (var client = new HttpClient())
 {
     client.SendAsync(
         new HttpRequestMessage(
             HttpMethod.Get, 
             "http://localhost:8080"
         )
     ).Wait();
 }
```

## Step 2\. Select testing strategy

The correct way to test web application as a unit is to interact with it the same way as user does using web browser. We need [Selenium.WebDriver](https://www.nuget.org/packages/Selenium.WebDriver/) (browser automation tool), [Selenium.Support](https://www.nuget.org/packages/Selenium.Support/) (support classes) and one of the browser drivers like [Selenium.WebDriver.IEDriver](https://www.nuget.org/packages/Selenium.WebDriver.IEDriver/) or [Selenium.WebDriver.ChromeDriver](https://www.nuget.org/packages/Selenium.WebDriver.ChromeDriver/). We also need [SpecFlow](https://www.nuget.org/packages/SpecFlow/) as testing framework.   
SpecFlow allows us to write test scenarios in [Gherkin](https://en.wikipedia.org/wiki/Cucumber_(software)) language which looks like this:

``` gherkin
 Feature: Authentication
     As a Heroes of Storm Administrator
     In order to feel that character control is in secure
     I want to have administration tool secured by username and password

 Scenario: Being able to authenticate
     Given there is are following users in the system
         | Name  | Password |
         | Megan | 123456   |
     When I navigate to "Listing" page
     Then I should be on "Login" page
     When I fill in controls as follows
         | Name     | Value  |
         | UserName | Megan  |
         | Password | 123456 |
     And I click "Sign In" button
     Then I should be on “Listing” page
```

For each line of this scenario SpecFlow will generate a function with **given, when** or **then** attribute that has regexp string. 

``` csharp
 [Then(@"I should see text ""(.*)""")]
 public void ShouldSeeText(string text)
 {
     var elements = BrowserDriver.FindElementsByXPath($".//*[text() = \"{text}\"]");
     Assert.That(elements.Any());
 }
 ```

As you can see, you can then use these functions to organize browser interaction.

## Step 3\. Mocking application components

![](/images/3-steps-integration-testing-aspnet-mvc/integration-testing-mock-component.png)

Having steps 1 and 2 done, you already can test your application. But this will be a System Testing or End-To-End testing because you have a real application and you are interacting with it as a real user. In this case you need to have all components working including database and third-party services integration. What if you want to pre-set some state or each individual testing scenario? Well. You will have to initiate state in database and you will have to do something with third-party service as well. This can be very difficult. So here is the way how to **mock components in working asp.net mvc application**.

We need to make a reference to [Deleporter](https://www.nuget.org/packages/Deleporter/) package in Web project as well as in test project. After this we need to add httpModule in Web.config file:

``` xml
 <modules>
     <add name="DeleporterServerModule" 
          type="DeleporterCore.Server.DeleporterServerModule, Deleporter" />
 </modules>
 ```

Also we need to configure this module

``` xml
<configSections>
     <section name="deleporter" 
              type="DeleporterCore.Configuration.DeleporterConfigurationSection, Deleporter"/>
 </configSections>
 <deleporter RemotingPort="38473" WebHostPort="8080" LoggingEnabled="true" />
```

This module opens tcp socket on port number 38473 if and only if application is running in "Debug" mode and on port 8080\. It is listening for remote commands to execute. And this is why we made web request in step 1, to initialize this module.

Now we need to make a reference to Deleporter in test project and now we can do:

``` csharp 
Deleporter.Run(() =>
 {
     var repository = new Mock<IGenericRepository<Account>>();
     repository.Setup(x => x.Entities).Returns(accounts.AsQueryable());
     Global.Container.Mock<IGenericRepository<Account>>(repository.Object);
 });
```

Deleporter will serialize the delegate, pass through tcp channel, deserialize it in web application and execute. Few important notes to make it happen:

*   All classes within the delegate should be serializable
*   All classes within the delegate should be referenced in web application. (If you are using mocking library then you should have a reference to it in web project too)
*   You should implement component replacement functionality for your IOC container.

In my case Global.Container is the reference to IOC container and Mock<T>() is the function that removes default component and replaces it with everything I want in runtime.

This is it. If you want to see all this in action please take a look at [this github repo](https://github.com/lAnubisl/WebApplicationsDevelopmentLessons/tree/master/Lesson%2019%20-%20Testing/Integration%20testing/ASP.NET%20MVC).