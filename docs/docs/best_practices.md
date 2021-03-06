# Best Practices for Developing Liquidart Applications

## Keep Dart Projects Separate

Because Dart is cross-platform, developers should avoid combining client application projects with Liquidart projects. Instead, use a single repository with an independent project for each facet of the system. When there are opportunities for code sharing between platforms (typically between Flutter and AngularDart), shared code can live in a dependency project in the same repository.

A typical directory structure for an multi-faceted application looks like this:

```
application_name/
  liquidart/
  flutter/
  angular/
  shared/
```

!!! Project Definition
    A project is a directory that contain a pubspec.yaml file and lib directory.

It is tempting to share your data model types between your server and client applications, but this falls apart for anything but the most simple of applications. There are enough behavioral differences between the four representations of your data model - in the database, on the server, on the wire (JSON), and on the client - that a single type will have a hard time encompassing. Instead, generate an OpenAPI specification with `liquidart document` and use one of the many open-source tools for generating client data model types.

## Use Test Driven Development (or something close to it)

In Liquidart, testing is a first-class citizen. The `liquidart_test` package has classes and methods for initializing and running an application for testing, making requests to that application, and verifying the responses. There is value to using tools like Postman or CURL to test proof of concept code, but the `liquidart_test` package is geared specifically for replacing these tools while retaining automated tests as the project grows.

An example test suite looks like this:

```dart
void main() {
  final harness = new Harness()..install();

  test("GET /endpoint returns 200 and a simple object", () async {
    final response = await harness.agent.get("/endpoint");
    expectResponse(response, 200, body: {"key": "value"});
  });
}
```

## Use a bin Script to Verify Assumptions

Keep a simple Dart script file in the `bin/` directory that imports your project. Use this file as a scratchpad to test exploratory code before committing to a test suite. Don't check this file into source control.

```dart
import 'package:myapp/myapp.dart';

Future main() async {
  var whatIsThis = await someYetToBeNamedUsefullyMethod();
  print("$whatIsThis");
}
```

## Create New Projects from a Template

Use `liquidart create` to create applications with the appropriate structure and boilerplate. There are templates for different kinds of applications; view these templates with `liquidart create list-templates`.

## Use a Debugger

A debugger allows you to stop execution of a running application at a particular line of code to verify variable values, and then continue to step through that code line by line. It can be used when running test suites or when running the application through the `bin/main.dart` script.

In IntelliJ IDEA, right-click on any file with a `main` function (which includes test suites) and select `Debug` option. Use breakpoints (by clicking on the gutter area to the left of the text editing area) to stop execution at a particular line before it is executed.

## Use the Suggested Project Directory Structure

See [Liquidart Project Structure](https://aldrinsartfactory.github.io/liquidart/application/structure/#liquidart-project-structure-and-organization).

## Pass Services to Controllers in entryPoint

Pass service objects to controllers in `entryPoint` and only pass the services the controller will use.

```dart
class AppChannel extends ApplicationChannel {
  GitHub githubService;
  PostgreSQLConnection databaseConnection;

  @override
  Future prepare() async {
    databaseConnection = new PostgreSQLConnection();
    githubService = new GitHub();
  }

  @override
  Controller get entryPoint {
    final router = new Router();

    router
      .route("/data")
      .link(() => new DBController(databaseConnection));

    router
      .route("/github")
      .link(() => new GitHubController(githubService));

    return router;
  }
}
```

Passing references like this allows for injecting dependencies that depend on the environment; e.g. in production, development or during tests. It also avoids tight coupling between the objects in your application.

Minimize the access a controller has to its dependencies; e.g. don't pass it a `StreamController` when it only needs `Sink` or a `Stream`.

## Use a Test Harness

A test harness initializes your application in a test suite. It has built in behavior that you can add to for things that are specific to your application. Documentation for using a test harness in your application is located [here](https://aldrinsartfactory.github.io/liquidart/testing/tests/).

## Use config.src.yaml

Use the convention of [config.src.yaml](https://aldrinsartfactory.github.io/liquidart/application/configure/) file to prevent configuration errors and inject test dependencies.

## Understand how Liquidart Uses Isolates

See more in [Application Structure](https://aldrinsartfactory.github.io/liquidart/application/structure/).

## Use ResourceController Subclasses

Subclassing [ResourceController](https://aldrinsartfactory.github.io/liquidart/http/resource_controller/) provides significant conveniences, safeties and behaviors used by the majority of an application's request handling logic. Prefer to use this class for non-middleware controllers.

## Keep ApplicationChannel Tidy

A `ApplicationChannel` should handle initialization, routing and nothing more. Consider moving non-initialization behavior into a service object in a separate file.

## Avoid Raw SQL Queries

Prefer to use the Liquidart ORM. It sends appropriate HTTP responses for different kinds of errors, validates input data and is ensures the queries match up with your data model.

## Use API Reference

Liquidart is an object oriented framework - behaviors are implemented by instances of some type. The types of objects, their properties and their behaviors all follow similar naming conventions to make the API more discoverable.

Many types in Liquidart have a prefix in common with related types. For example, types like `AuthServer`, `AuthServerDelegate` and `AuthCode` are all related because they deal with authentication and authorization. Methods are named consistently across classes (e.g, `asMap` is a common method name).

When looking for a solution, look at the [API reference](https://pub.dev/documentation/liquidart/latest/) for the objects you have access to. These objects may already have the behavior you wish to implement or have a reference to an object with that behavior.

## Use try-catch Sparingly

All request handling code is wrapped in a try-catch block that will interpret exceptions and errors and return meaningful HTTP responses. Unknown exceptions will yield a 500 Server Error response. In general, you do not need to use try-catch unless you want a different HTTP response than the one being returned for the exception.

Code that throws an exception during initialization should not be caught if the error is fatal to the application launching successfully.