![](https://heatbadger.now.sh/github/readme/contributte/apitte-core/?deprecated=1)

<p align=center>
    <a href="https://bit.ly/ctteg"><img src="https://badgen.net/badge/support/gitter/cyan"></a>
    <a href="https://bit.ly/cttfo"><img src="https://badgen.net/badge/support/forum/yellow"></a>
    <a href="https://contributte.org/partners.html"><img src="https://badgen.net/badge/sponsor/donations/F96854"></a>
</p>

<p align=center>
    Website 🚀 <a href="https://contributte.org">contributte.org</a> | Contact 👨🏻‍💻 <a href="https://f3l1x.io">f3l1x.io</a> | Twitter 🐦 <a href="https://twitter.com/contributte">@contributte</a>
</p>

## Disclaimer

| :warning: | This project is no longer being maintained. Please use [contributte/apitte](https://github.com/contributte/apitte).|
|---|---|

| Composer | [`apitte/core`](https://packagist.org/packages/apitte/core) |
|---| --- |
| Version | ![](https://badgen.net/packagist/v/apitte/core) |
| PHP | ![](https://badgen.net/packagist/php/apitte/core) |
| License | ![](https://badgen.net/github/license/contributte/apitte-core) |

## Usage

To install the latest version of `apitte/core` use [Composer](https://getcomposer.org).

```bash
composer require apitte/core
```

## Documentation

Core library of Apitte API framework.

### Setup

Install core

```bash
composer require apitte/core
```

Register DI extension

```yaml
extensions:
    api: Apitte\Core\DI\ApiExtension

api:
    debug: %debugMode%
    catchException: true # Sets if exception should be catched and transformed into response or rethrown to output (debug only)
```

Create entry point

```php
// www/index.php

use Apitte\Core\Application\IApplication;
use App\Bootstrap;

require __DIR__ . '/../vendor/autoload.php';

Bootstrap::boot()
    ->createContainer()
    ->getByType(IApplication::class)
    ->run();
```

#### Usage in combination with nette application

```php
// www/index.php

use Apitte\Core\Application\IApplication as ApiApplication;
use App\Bootstrap;
use Nette\Application\Application as UIApplication;

require __DIR__ . '/../vendor/autoload.php';

$isApi = substr($_SERVER['REQUEST_URI'], 0, 4) === '/api';
$container = Bootstrap::boot()->createContainer();

if ($isApi) {
    $container->getByType(ApiApplication::class)->run();
} else {
    $container->getByType(UIApplication::class)->run();
}
```

### Endpoints

Endpoint is representation of a unique url (like a `/api/v1/users`) and one or multiple operations (HTTP methods).

In our case endpoint is implemented as a controller method.

#### Controllers

Create base controller with root path to your api

- controller must implement `Apitte\Core\UI\Controller\IController`

```php
namespace App\Api\V1\Controllers;

use Apitte\Core\Annotation\Controller\Path;
use Apitte\Core\UI\Controller\IController;

/**
 * @Path("/api/v1")
 */
abstract class BaseV1Controller implements IController
{
}
```

Create an endpoint

- Controller must have annotation `@Path()` and be registered as service
- Method must have annotations `@Path()` and `@Method()`

```yaml
services:
    - App\Api\V1\Controllers\UsersController
```

```php
namespace App\Api\V1\Controllers;

use Apitte\Core\Annotation\Controller\Method;
use Apitte\Core\Annotation\Controller\Path;
use Apitte\Core\Http\ApiRequest;
use Apitte\Core\Http\ApiResponse;
use Nette\Utils\Json;

/**
 * @Path("/users")
 */
class UsersController extends BaseV1Controller
{

    /**
     * @Path("/")
     * @Method("GET")
     */
    public function index(ApiRequest $request, ApiResponse $response): ApiResponse
    {
        // This is an endpoint
        //  - its path is /api/v1/users/
        //  - it should be available on address example.com/api/v1/users/

        $response = $response->writeBody(Json::encode([
            [
                'id' => 1,
                'firstName' => 'John',
                'lastName' => 'Doe',
                'emailAddress' => 'john@doe.com',
            ],
            [
                'id' => 2,
                'firstName' => 'Elon',
                'lastName' => 'Musk',
                'emailAddress' => 'elon.musk@spacex.com',
            ],
        ]));

        return $response;
    }

}
```

#### List of annotations / attributes

> You can use seamless PHP 8 attributes.

`@Id`
  - Must consist only of following characters: `a-z`, `A-Z`, `0-9`, `_`
  - Not used by Apitte for anything, it may just help you identify, group, etc. your endpoints

`@Path`
  - Must consist only of following characters: `a-z`, `A-Z`, `0-9`, `-_/`
  - The `@Path` annotation can be used on:
    - abstract controller to define a group path for multiple controllers (e.g. `example.com/v1/...`)
    - final controller to define a path for that particular controller (e.g. `example.com/v1/users`)
    - method to define a path for a specific endpoint
  - This hierarchy is then used to build the schema and make routing possible.

`@Method`
  - Allowed HTTP method for endpoint
  - GET, POST, PUT, OPTION, DELETE, HEAD
  - `@Method("GET")`
  - `@Method({"POST", "PUT"})`
  - Defined on method

`@Tag`
  - Used by OpenApi
  - Could by also used by your custom logic
  - `@Tag(name="name")`
  - `@Tag(name="string", value="string|null")`
  - Defined on class and method

### Mapping

Validate and map data from request and map data to response.

#### Setup

```yaml
api:
    plugins:
        Apitte\Core\DI\Plugin\CoreMappingPlugin:
```

Ensure you have also decorator plugin registered, mapping is implemented by decorators.

#### RequestParameters

Validate request parameters and convert them to correct php datatype.

```php
namespace App\Api\V1\Controllers;

use Apitte\Core\Annotation\Controller\Method;
use Apitte\Core\Annotation\Controller\Path;
use Apitte\Core\Annotation\Controller\RequestParameters;
use Apitte\Core\Annotation\Controller\RequestParameter;
use Apitte\Core\Http\ApiRequest;
use Apitte\Core\Http\ApiResponse;

/**
 * @Path("/users")
 */
class UsersController extends BaseV1Controller
{

    /**
     * @Path("/{id}")
     * @Method("GET")
     * @RequestParameters({
     *      @RequestParameter(name="id", type="int", description="My favourite user ID")
     * })
     */
    public function detail(ApiRequest $request): ApiResponse
    {
        /** @var int $id Perfectly valid integer */
        $id = $request->getParameter('id');

        // Return response with error or user
    }

}
```

#### RequestBody

Use value objects for request body mapping:

```php
namespace App\Api\Entity\Request;

use Apitte\Core\Mapping\Request\BasicEntity;

final class UserFilter extends BasicEntity
{

    /**  @var int */
    public $userId;

    /**  @var string */
    public $email;

}
```

```php
/**
 * @Path("/filter")
 * @Method("GET")
 * @RequestBody(entity="App\Api\Entity\Request\UserFilter")
 */
public function filter(ApiRequest $request)
{
    /** @var UserFilter $entity */
    $entity = $request->getEntity();
}
```

### Decorators

Decorators are used for transformations of request before it is passed into endpoint and for transformations of response after it is returned from endpoint.

#### Setup

```yaml
api:
    plugins:
        Apitte\Core\DI\Plugin\CoreDecoratorPlugin:
```

#### Register decorators

```yaml
services:
    decorator.request.authentication:
        class: App\Api\Decorator\ExampleResponseDecorator
        tags: [apitte.core.decorator: [priority: 50]]
```

### Router

Checks if an endpoint from schema matches request.

#### SimpleRouter

Default implementation of router which matches endpoint by URI and by HTTP method.

Requires each endpoint to have an unique combination of URI and HTTP method.

If an endpoint for given URI exists but not for given HTTP method then `405 Method Not Allowed` is returned.

### Errors

To display errors to user use our prepared `ApiException`, specifically:

- `ClientErrorException` for user errors (400-499)
- `ServerErrorException` for server errors (500-599)

#### SimpleErrorHandler

Default error handler transforms error into json response:

```json
{
  "status": "error",
  "code": 500,
  "message": "Application encountered an internal error. Please try again later.",
  "context": []
}
```

### Examples

- https://github.com/contributte/playground (playground)
- https://contributte.org/examples.html (more examples)

## Version

| State       | Version | Branch   | Nette | PHP     |
|-------------|---------|----------|-------|---------|
| stable      | `^0.8`  | `master` | 3.0+  | `>=7.3` |
| stable      | `^0.5`  | `master` | 2.4   | `>=7.1` |

## Development

This package was maintained by these authors.

<a href="https://github.com/f3l1x">
  <img width="80" height="80" src="https://avatars2.githubusercontent.com/u/538058?v=3&s=80">
</a>

-----

Consider to [support](https://contributte.org/partners.html) **contributte** development team.
Also thank you for using this package.
