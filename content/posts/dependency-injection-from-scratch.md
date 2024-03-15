---
title: "Inside Dependency Injection: building DI from scratch"
date: 2023-03-15T23:15:57+01:00
draft: false
tags: [ "dependency injection", "PHP", "software design" ]
description: "Most frameworks nowadays use Dependency Injection. In this article I will get inside the basics how a DI works by building one from scratch"
---

# What is a Dependency Injection (DI)

Most frameworks nowadays use dependency injection.
In this article, I will get inside the basics how a DI works by building one in PHP from scratch.

A Dependency Injection system is used to solve the issue of having to
(re-)using instances of services around the application by providing a central way to access each and every service.
They also take care of making all required dependencies available just in time — therefore the name.

# Road to service container

A typical service you will need again and again in an application is your database connection.
You want to configure and establish it once and then be able to reuse it all the time.
Therefore this will be my example service for the following examples

###Globals

Most programming languages have different scopes for their variables,
with most languages having a concept of a global variable.
Depending on the language you have to either define it explicitly as global
(for example in PHP) or it will run implicitly available in sub contexts (like in JS).

```php
global $DB = new DatabaseConnection('localhost', 3306, 'user', 'password');

function myFunction() {
    $result = $DB->query('SELECT * FROM foo');
}
```

There are several obvious issues with this approach:

1. As this is a variable, I can override it at any time in any part of my application
   breaking havoc if my `$DB` now not the same database connection anymore.
   Maybe two devs where developing their features and both needed some database and used the same variable.
   It worked as long as they did not merge, but now you cannot tell which connection is behind that descriptive
   variable.

2. It is global by its nature, and every part of the application can use it.
   I cannot trace it uses through the application because one part of the application might define its own **local
   ** `$DB`.

3. I always have to instantiate it even if I don't need it. This is a waste of resources.

## Singletons

Singletons are the result of the mentioned issues.
They are a software pattern
that aims to have a single instance of a given object that will be reused for every consecutive use case.
To achieve this task there is always a method to get the same instance again, and if none exists yet they will create
it.

Pure function:

```php
function getDbInstance(): DatabaseConnection {
    static $instance;
    
    if (!$instance) {
        $instance = new DatabaseConnection('localhost', 3306, 'user', 'password');
    }
    
    return $instance;
}

$db = getDbInstance();
```

OOP

```php
class DatabaseConnection {
    private static $instance = null;
    
    // explicit constructor to prevent `new DatabaseConnection()`
    protected function __construct() {}
    
    public static function getInstance(): self {
        if (!self::$instance) {
            self::$instance = new self('localhost', 3306, 'user', 'password');
        }
        return self::$instance;
    }
}

$db = DatabaseConnection::getInstance();
```

As classes and functions are (in most languages) immutable,
you still have them in global space, but you can be sure that they will return the same object every time.
On the flip side:
you have to explicitly make a class or service a singleton or provide a singleton wrapper, which means extra work.
Also,
you now have either the configuration global instead or weave it
(like in my example) directly into your service, which makes it harder to reuse.

Something we haven't talked about yet is testing.
Considering the example with our database connection,
we normally do not want to call our production database
and therefore need a different connection with different configuration.
In this case, the aforementioned configuration within our service is a big no-go.

## Containers

Containers are a standardized way to store things.
A standard shipment container is typically 10, 20 or 40 ft long, with a width of 8ft and height of 8.5ft.
They have doors and can be stacked however the operator likes.
What's inside does not matter — may it be cheap chinese fast fashion, bananas or cocaine (or bananas and cocaine).

In the programming world we also use lots of containers for storing things. One of the kind is the _service container_.

```php
class ServiceContainer {
    private static $instance;
    private array $services = [];
    
    public static function getInstance(): self {
        if (!self::$instance) {
            self::$instance = new self('localhost', 3306, 'user', 'password');
        }
        return self::$instance;
    }
    
    public function set(string $name, mixed $service) {
       $this->services[$name] = $service;
   }
    
    public function get(string $name) {
       if (!isset($this->services[$name])) {
            throw new RuntimeException('Unknown service ' . $name);
       }
       return $this->services[$name];
   }
}

// ...

// For now: use singleton to make service container available everywhere
// configuration
$container = ServiceContainer::getInstance();
$container->set('MyVeryCoolSecreteDatabaseConnection', new DatabaseConnection('localhost', 42000, 'user', 'password'));
$container->set(DatabaseConnection::class, new DatabaseConnection('127.0.0.1', 3306, 'user', 'password'));

// ...
// In my app now I can do following
$db = ServiceContainer::getInstance()->get('MyVeryCoolSecreteDatabaseConnection');
$result = $qb->query('SELECT * FROM foo');


$db2 = ServiceContainer::getInstance()->get(DatabaseConnection::class);
$result = $qb2->query('SELECT * FROM foo');
```

So far, so good.
To make things easy it is common practice to use the fully qualified class name
(FQCN)
of the service you want to share although no-one will prevent you to provide a service under whatever name you like.
Hence, you can even have several instances of a service like different database connections
using the same `DatabaseConnection` class.

When testing, we sometimes we do not want to have a connection to a database at all
but instead use a mocked service that always returns the same, predefined values.
What do we now?
Well, thanks to `interfaces` we can now do the following:

```php
interface DatabaseConnectionInterface {
    public function query(string $sql): array;
}

class DatabaseConnection implements DatabaseConnectionInterface {}
class MockDatabaseConnection implements DatabaseConnectionInterface {}

// CONFIGURATION
// In productionx
$container->set(DatabaseConnectionInterface::class, new DatabaseConnection('127.0.0.1', 3306, 'user', 'password'));

// For testing
$container->set(DatabaseConnectionInterface::class, new MockDatabaseConnection());

// ...
// In our application: we use whatever implementation is available
$db = ServiceContainer::getInstance()->get(DatabaseConnectionInterface::class);
$result = $qb->query('SELECT * FROM foo');
```

## Factories

In software design we use the [factory pattern](https://refactoring.guru/design-patterns/factory-method)
to create new instances of a specific component.
This can be a `WindowFactory`
that creates a new `Window` in your UI with specific configurations or like in our example a new database connection.

If you paid a little bit attention one goal of the `Singleton` pattern was the instantiation when it is first needed
(`getInstance()`) which we lost in our previous example with our service container.
But fear not: there is an easy fix by extending out

```php
class ServiceContainer {
    private array $services = [];
    
    // this is new
    private array $factories = []:
    
    public function set(string $name, mixed $service) {
       $this->services[$name] = $service;
   }
   
    public function setFactory(string $name, callable $factory) {
        $this->factories[$name] = $factory;
    }
    
    public function get(string $name) {
       if (!isset($this->services[$name])) {
            // this is new
            if (!isset($this->factories[$name])) {
                throw new RuntimeException('Unknown service ' . $name);
            }
            $this->set($name, $factory($this, $name));
       }
       return $this->services[$name];
   }
}
```

With this new extended `ServiceContainer` we have two ways to get a service:
it is either given already preconfigured or we have a factory-callback that will return the required instance.

```php
$serviceContainer->setFactory(DatabaseConnectionInterface::class, fn() => new DatabaseConnection('127.0.0.1', 3306, 'user', 'password')));

// ...

// create instance only when needed
$db = $serviceContainer->get(DatabaseConnectionInterface::class);
$db->query(...);
```

### Using factories recursive

So now we can configure our database connection in one place
and can reuse it somewhere else in our application without having to access anything but our `ServiceContainer`.
But we can still do better!
Let's use our new factory functionality
to create dependent services like a `DocomentRepository` that is fetching documents from our database.
Maybe you have seen that we passed our `ServiceContainer` to our factory although its factory did not use it?

```php
class DocumentRepository {
    public function __construct(private DatabaseConnectionInterface $db) {}
    
    public function find($id): Document {
        return $this->db->query('SELECT * FROM document WHERE id=' . $id);
    }
}

$serviceContainer->setFactory(DatabaseConnectionInterface::class, fn() => new DatabaseConnection('127.0.0.1', 3306, 'user', 'password')));
$serviceContainer->setFactory(
    DocumentRepository::class,
    fn(ServiceContainer $container) => new DocumentRepository($container->get(DatabaseConnectionInterface::class))
);

// ...

$repository = $serviceContainer->get(DocumentRepository::class);
$repository->find(1);
```

So what are we doing here?
Thanks to our service definition we have two services defined via their factory methods,
with the `DocumentRepository` being dependent upon the `DatabaseConnection`.
When we first request our `DocumentRepository` its factory method will be called.
This factory method itself now uses the `ServiceContainer`
and triggers the creation of the `DatabaseConnection`
to provide its instance to the constructor of the `DocumentRepository`.

# Conclusion

Thanks to our new dependency injection, we are now able to

- configure our services in a single space in our code
- still be able to initiate our services lazily and only if needed
- decouple services and their dependencies (as long as we use interfaces) which enhances our ability to mock without
  much effort a lot
- leave our global variable scope clean as our only entry point is our `ServiceContainer` which encapsulates everything
  else — still while not knowing anything over any service it manages


## So what's next?
In a future blog post, I want to elaborate on the idea and introduce some advanced features like an `alias` system,
structure for systematic creation & configuration
as well as so called `autowiring`
to reduce the work we have to put in to create new service instances that are only dependent on other services.