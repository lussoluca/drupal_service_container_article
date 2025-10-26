# Drupal service container deep dive part 1: tags, compiler passes, service providers and autoconfiguration

The evolution of Drupal core has increasingly embraced modern PHP standards and robust object-oriented programming (OOP) principles, driven largely by its integration of **Symfony components**. Central to this modernization is the **Service Container**, which acts as the core registry responsible for service instantiation, dependency resolution, and lifecycle management.

A **service** is inherently a specialized, reusable, and stateless object designed to perform a single, well-defined task, such as ConfigFactory for configuration access or EntityTypeManager for entity access.  

The philosophy underpinning service usage is the promotion of loose coupling and adherence to the Dependency Inversion Principle (DIP). A class using a service should request it from the container rather than manually instantiating it with the `new` keyword. This arrangement enables developers to swap implementations without altering consuming code.  

In this series of articles, we will explore the Drupal service container in depth, focusing on its features, capabilities, and best practices for leveraging it effectively in Drupal module development.

* **Part 1** will cover service tags, compiler passes, service providers, and autoconfiguration, laying the groundwork for understanding how services are defined and managed within the container.
* **Part 2** will delve into aliases, autowiring, and name arguments, demonstrating how to resolve and inject dependencies based on type hints, reducing boilerplate code and enhancing maintainability.
* **Part 3** will cover service collectors, which aggregate multiple services into a single service for streamlined access.
* **Part 4** will explore factories, which provide a mechanism for creating services with complex initialization logic.
* **Part 5** will discuss backend overrides, enabling developers to customize service implementations for specific environments or use cases.
* **Part 6** will examine advanced features of the Drupal service container, such as service decoration and lazy loading.

## Anatomy of a Drupal service

A Drupal service is defined within a module's `MODULE.services.yml` file:

```yml
# in modules/contrib/webprofiler/webprofiler.services.yml
services:
    webprofiler.matcher.exclude_path:
        class: Drupal\webprofiler\RequestMatcher\WebprofilerRequestMatcher
        arguments: ['@path.matcher', '@config.factory', 'exclude_paths']

# in core/core.services.yml
services:
    path.matcher:
        class: Drupal\Core\Path\PathMatcher
        arguments: ['@config.factory', '@current_route_match']
    config.factory:
        class: Drupal\Core\Config\ConfigFactory
    current_route_match:
        class: Drupal\Core\Routing\CurrentRouteMatch
        arguments: ['@request_stack']
    request_stack:
        class: Symfony\Component\HttpFoundation\RequestStack
```

When code asks for the `webprofiler.matcher.exclude_path` service, the Drupal service container first builds all the required services and finally builds an instance of the `WebprofilerRequestMatcher` class by injecting all the dependencies in the `__construct` method:

```php
namespace Drupal\webprofiler\RequestMatcher;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Path\PathMatcherInterface;
use Symfony\Component\HttpFoundation\RequestMatcherInterface;

class WebprofilerRequestMatcher implements RequestMatcherInterface {

  public function __construct(
    private readonly PathMatcherInterface $pathMatcher,
    private readonly ConfigFactoryInterface $configFactory,
    private readonly string $configuration,
  ) {...}
}
```

The resolution graph looks like this:

```mermaid
graph TD
    B[current_route_match] --> A[request_stack];
    D[path.matcher] --> C[config.factory];
    D --> B;
    E[webprofiler.matcher.exclude_path] --> D;
    E --> C;
```

So Drupal first builds the `request_stack` service, then the `current_route_match` service (which depends on `request_stack`), then the `config.factory` service, then the `path.matcher` service (which depends on both `config.factory` and `current_route_match`), and finally the `webprofiler.matcher.exclude_path` service (which depends on both `path.matcher` and `config.factory`).

Notice that the third argument of the `webprofiler.matcher.exclude_path` service is a string (`exclude_paths`). The service container supports injecting scalar values like strings, integers, and booleans as arguments to services:

* `@` prefix indicates a service reference.
* `%` prefix indicates a parameter reference.
* `@?` indicates an optional service reference.
* No prefix indicates a literal value.

```yml
parameters:
  param1: 'some string'

services:
  some_service1:
    class: Some\Class\Name1

  some_service2:
    class: Some\Class\Name2
    arguments: ['@some_service1', '%param1%', 42, true, 'another string', '@?some_service3']
```

and the code for `some_service2` would look like this:

```php
namespace Some\Class;

class Name2 {
  public function __construct(
    private readonly Name1 $someService1,
    private readonly string $param1,
    private readonly int $param2,
    private readonly bool $param3,
    private readonly string $param4,
    private readonly ?Name3 $someService3 = null,
  ) {...}
}
```

## Service Tags

Service tags are metadata annotations that can be applied to service definitions within the Drupal service container. They provide a way to categorize and group services based on their functionality or purpose. Tags are particularly useful for enabling features like event subscribers, plugins, and other extensible components.

When a service is tagged, it can be easily identified and retrieved by other parts of the system that need to interact with it. For example, event subscribers are tagged with the `event_subscriber` tag, allowing the event dispatcher to automatically discover and register them.   

## Compiler Passes

Compiler passes are a mechanism that allows developers to modify the service container during its compilation phase. They provide a way to programmatically alter service definitions, add or remove services, and manipulate tags before the container is fully built.

Compiler passes are particularly useful for implementing complex service registration logic, such as dynamically adding services based on configuration or other runtime conditions. They are executed in a specific order during the container compilation process, allowing for fine-grained control over the final service definitions.

## Service Providers

Service providers are classes that encapsulate the logic for registering services within the Drupal service container. They provide a structured way to define and organize service definitions, making it easier to manage dependencies and maintain a clean codebase.

Service providers typically implement the `ServiceProviderInterface` and define a `register` method where services are added to the container. This approach promotes modularity and separation of concerns, as each service provider can focus on a specific set of related services.

## Autoconfiguration

Autoconfiguration is a powerful feature of the Drupal service container that allows services to be automatically tagged and configured based on their class definitions. This reduces the need for manual configuration and helps maintain a clean and organized service definition.

When a service is defined in the container, the autoconfiguration process analyzes the class and applies the appropriate tags and configuration options. For example, if a service implements a specific interface or extends a particular base class, the container can automatically apply the relevant tags to the service definition.

Autoconfiguration is particularly useful for module developers, as it streamlines the process of registering services and ensures that they are properly configured without requiring extensive boilerplate code.

# Drupal service container deep dive part 2: aliases, autowiring and name arguments

## Aliases

Aliases provide a way to reference services using alternative names. This is particularly useful when you want to swap out implementations or provide a more descriptive name for a service. For example, you might create an alias for a logging service to refer to it as `logger.channel.custom`.

##  Autowiring

Autowiring allows you to manage services in the container with minimal configuration. It reads the type-hints on your constructor (or other methods) and automatically passes the correct services to each method.

> Thanks to Symfony's compiled container, there is no runtime overhead for using autowiring.

## Name Arguments

Name arguments allow you to specify which service should be injected based on a string identifier. This is useful when multiple services implement the same interface, and you need to distinguish between them. For example, if you have multiple payment gateway services implementing a `PaymentGatewayInterface`, you can use name arguments to specify which gateway to inject into a class.

# Drupal service container deep dive part 3: service collectors

# Drupal service container deep dive part 4: factories

# Drupal service container deep dive part 5: backend overrides

# Drupal service container deep dive part 6: advanced features

* service visibility
* service shared vs. non-shared services
* service decoration
* service properties
* lazy loading (https://symfony.com/doc/current/service_container/service_closures.html)
* abstract services
* abstract service arguments
* tagged iterator arguments
* calls
* service locators
* configurators
* deprecated services
