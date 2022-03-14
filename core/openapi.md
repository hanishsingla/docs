# OpenAPI Specification Support (formerly Swagger)

API Platform natively support the [OpenAPI](https://www.openapis.org/) API specification format.

![Screenshot](../distribution/images/swagger-ui-1.png)

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/open-api-spec?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="OpenAPI screencast"><br>Watch the OpenAPI screencast</a></p>

The specification of the API is available at the `/docs.json` path.
By default, OpenAPI v3 is used.
You can also get an OpenAPI v3-compliant version thanks to the `spec_version` query parameter: `/docs.json?spec_version=3`

It also integrates a customized version of [Swagger UI](https://swagger.io/swagger-ui/) and [ReDoc](https://rebilly.github.io/ReDoc/), some nice tools to display the
API documentation in a user friendly way.

## Using the OpenAPI Command

You can also dump an OpenAPI specification for your API.

OpenAPI, JSON format:

```console
docker-compose exec php \
    bin/console api:openapi:export
```

OpenAPI, YAML format:

```console
docker-compose exec php \
    bin/console api:openapi:export --yaml
```

Create a file containing the specification:

```console
docker-compose exec php \
    bin/console api:openapi:export --output=swagger_docs.json
```

If you want to use the old OpenAPI v2 (Swagger) JSON format, use:

```console
docker-compose exec php \
    bin/console api:swagger:export
```

## Overriding the OpenAPI Specification

Symfony allows to [decorate services](https://symfony.com/doc/current/service_container/service_decoration.html), here we
need to decorate `api_platform.openapi.factory`.

In the following example, we will see how to override the title of the Swagger documentation and add a custom filter for
the `GET` operation of `/foos` path.

```yaml
# api/config/services.yaml
    App\OpenApi\OpenApiFactory:
        decorates: 'api_platform.openapi.factory'
        arguments: [ '@App\OpenApi\OpenApiFactory.inner' ]
        autoconfigure: false
```

```php
<?php
namespace App\OpenApi;

use ApiPlatform\Core\OpenApi\Factory\OpenApiFactoryInterface;
use ApiPlatform\Core\OpenApi\OpenApi;
use ApiPlatform\Core\OpenApi\Model;

class OpenApiFactory implements OpenApiFactoryInterface
{
    private $decorated;

    public function __construct(OpenApiFactoryInterface $decorated)
    {
        $this->decorated = $decorated;
    }

    public function __invoke(array $context = []): OpenApi
    {
        $openApi = $this->decorated->__invoke($context);
        $pathItem = $openApi->getPaths()->getPath('/api/grumpy_pizzas/{id}');
        $operation = $pathItem->getGet();

        $openApi->getPaths()->addPath('/api/grumpy_pizzas/{id}', $pathItem->withGet(
            $operation->withParameters(array_merge(
                $operation->getParameters(),
                [new Model\Parameter('fields', 'query', 'Fields to remove of the output')]
            ))
        ));

        $openApi = $openApi->withInfo((new Model\Info('New Title', 'v2', 'Description of my custom API'))->withExtensionProperty('info-key', 'Info value'));
        $openApi = $openApi->withExtensionProperty('key', 'Custom x-key value');
        $openApi = $openApi->withExtensionProperty('x-value', 'Custom x-value value');

        return $openApi;
    }
}
```

The impact on the swagger-ui is the following:

![Swagger UI](core/images/swagger-ui-modified.png)

## Using the OpenAPI and Swagger Contexts

Sometimes you may want to change the information included in your OpenAPI documentation.
The following configuration will give you total control over your OpenAPI definitions:

[codeSelector]

```php
<?php
// api/src/Entity/Product.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\ApiProperty;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

#[ORM\Entity]
#[ApiResource]
class Product // The class name will be used to name exposed resources
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    /**
     * @param string $name A name property - this description will be available in the API documentation too.
     *
     */
    #[ORM\Column]
    #[Assert\NotBlank]
    #[ApiProperty(
        openapiContext: [
            'type' => 'string',
            'enum' => ['one', 'two'],
            'example' => 'one'
        ]
    )]
    public string $name;

    #[ORM\Column(type: "datetime")] 
    #[Assert\DateTime]
    #[ApiProperty(
        openapiContext: [
            'type' => 'string',
            'format' => 'date-time'
        ]
    )]
    public $timestamp;

    // ...
}
```

```yaml
# api/config/api_platform/resources.yaml
resources:
    App\Entity\Product:
      properties:
        name:
          attributes:
            openapi_context:
              type: string
              enum: ['one', 'two']
              example: one
        timestamp:
          attributes:
            openapi_context:
              type: string
              format: date-time
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<properties xmlns="https://api-platform.com/schema/metadata/properties"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/properties
           https://api-platform.com/schema/metadata/properties.xsd">
    <property resource="App\Entity\Product" name="name">
        <openapiContext>
            <values>
                <value name="type">type</value>
                <value name="enum">
                    <values>
                        <value>one</value>
                        <value>two</value>
                    </values>
                </value>
                <value name="example">one</value>
            </values>
        </openapiContext>
    </property>
    <property resource="App\Entity\Product" name="timestamp">
        <openapiContext>
            <values>
                <value name="type">string</value>
                <value name="format">date-time</value>
            </values>
        </attribute>
    </property>
</properties>
```

[/codeSelector]

This will produce the following Swagger documentation:

```json
"components": {
    "schemas": {
        "GrumpyPizza:jsonld": {
            "type": "object",
            "description": "",
            "properties": {
                "@context": {
                    "readOnly": true,
                    "type": "string"
                },
                "@id": {
                    "readOnly": true,
                    "type": "string"
                },
                "@type": {
                    "readOnly": true,
                    "type": "string"
                },
                "timestamp": {
                    "type": "string",
                    "format": "date-time"
                },
                "id": {
                    "readOnly": true,
                    "type": "integer"
                },
                "name": {
                    "type": "string",
                    "enum": [
                        "one",
                        "two"
                    ],
                    "example": "one"
                }
            }
        }
    }
}
```

To pass a context to the OpenAPI **v2** generator, use the `swaggerContext` attribute (notice the prefix: `swagger` instead of `openapi`).

## Changing the Name of a Definition

API Platform generates a definition name based on the serializer `groups` defined
in the (`de`)`normalizationContext`. It's possible to override the name
thanks to the `swagger_definition_name` option:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Post;

#[ApiResource]
#[Post(denormalizationContext: ['groups' => ['user:read'], 'swagger_definition_name' => 'Read'])]
class User
{
    // ...
}
```

It's also possible to re-use the (`de`)`normalizationContext`:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Post;

#[ApiResource]
#[Post(denormalizationContext: [User::API_WRITE])]
class User
{
    const API_WRITE = [
        'groups' => ['user:read'],
        'swagger_definition_name' => 'Read',
    ];
}
```

## Changing Operations in the OpenAPI Documentation

You also have full control over both built-in and custom operations documentation.

[codeSelector]

```php
<?php
// api/src/Entity/Rabbit.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Post;
use App\Controller\RandomRabbit;

#[ApiResource]
#[Post(
    name: 'create_rabbit', 
    uriTemplate: '/rabbit/create', 
    controller: RandomRabbit::class, 
    openapiContext: [
        'summary' => 'Create a rabbit picture', 
        'description' => '# Pop a great rabbit picture by color!\n\n![A great rabbit](https://rabbit.org/graphics/fun/netbunnies/jellybean1-brennan1.jpg)', 
        'requestBody' => [
            'content' => [
                'application/json' => [
                    'schema' => [
                        'type' => 'object', 
                        'properties' => [
                            'name' => ['type' => 'string'], 
                            'description' => ['type' => 'string']
                        ]
                    ], 
                    'example' => [
                        'name' => 'Mr. Rabbit', 
                        'description' => 'Pink Rabbit'
                    ]
                ]
            ]
        ]
    ]
)]
class Rabbit
{
    // ...
}
```

```yaml
resources:
  App\Entity\Rabbit:
    operations:
      create_rabbit:
        class: ApiPlatform\Metadata\Post
        path: '/rabbit/create'
        controller: App\Controller\RandomRabbit
        openapiContext:
          summary: Random rabbit picture
          description: >
            # Pop a great rabbit picture by color!

            ![A great rabbit](https://rabbit.org/graphics/fun/netbunnies/jellybean1-brennan1.jpg)

          requestBody:
            content:
              application/json:
                schema:
                  type: object
                  properties:
                    name: { type: string }
                    description: { type: string }
                example:
                  name: Mr. Rabbit
                  description: Pink rabbit

```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<resources xmlns="https://api-platform.com/schema/metadata/resources"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources
        https://api-platform.com/schema/metadata/resources.xsd">
    <resource class="App\Entity\Rabbit">
        <operations>
            <operation class="ApiPlatform\Metadata\Post" name="create_rabbit" uriTemplate="/rabbit/create"
                       controller="App\Controller\RandomRabbit">
                <openapiContext>
                    <values>
                        <value name="summary">Create a rabbit picture </value>
                        <value name="description"># Pop a great rabbit picture by color!!
    
    ![A great rabbit](https://rabbit.org/graphics/fun/netbunnies/jellybean1-brennan1.jpg)</value>
                        <value name="content">
                            <values>
                                <value name="application/json">
                                    <values>
                                        <value name="schema">
                                            <values>
                                                <value name="type">object</value>
                                                <value name="properties">
                                                    <values>
                                                        <value name="name">
                                                            <values>
                                                                <value name="type">string</value>
                                                            </values>
                                                        </value>
                                                        <value name="description">
                                                            <values>
                                                                <value name="type">string</value>
                                                            </values>
                                                        </value>
                                                    </values>
                                                </value>
                                            </values>
                                        </value>
                                    </values>
                                </value>
                            </values>
                        </value>
                    </values>
                </openapiContext>
            </operation>
        </operations>
    </resource>
</resources>
```

[/codeSelector]

![Impact on Swagger UI](../distribution/images/swagger-ui-2.png)

## Changing the Location of Swagger UI

Sometimes you may want to have the API at one location, and the Swagger UI at a different location. This can be done by disabling the Swagger UI from the API Platform configuration file and manually adding the Swagger UI controller.

### Disabling Swagger UI or ReDoc

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    # ...
    enable_swagger_ui: false
    enable_re_doc: false
```

### Manually Registering the Swagger UI Controller

```yaml
# app/config/routes.yaml
swagger_ui:
    path: /docs
    controller: api_platform.swagger.action.ui
```

Change `/docs` to the URI you wish Swagger to be accessible on.

## Using a custom Asset Package in Swagger UI

Sometimes you may want to use a different [Asset Package](https://symfony.com/doc/current/reference/configuration/framework.html#packages) for the Swagger UI. In this way you'll have more fine-grained control over the asset url generations. This is useful i.e. if you want to use different base path, base url or asset versioning strategy.

Specify a custom asset package name:

```yaml
# config/packages/api_platform.yaml
api_platform:
    asset_package: 'api_platform'
```

Set or override asset properties per package:

```yaml
# config/packages/framework.yaml
framework:
    # ...
    assets:
        base_path: '/custom_base_path' # the default
        packages:
            api_platform:
                base_path: '/'
```

## Overriding the UI Template

As described [in the Symfony documentation](https://symfony.com/doc/current/templating/overriding.html), it's possible to override the Twig template that loads Swagger UI and renders the documentation:

```twig
{# templates/bundles/ApiPlatformBundle/SwaggerUi/index.html.twig #}
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>{% if title %}{{ title }} {% endif %}My custom template</title>
    {# ... #}
</html>
```

You may want to copy the [one shipped with API Platform](https://github.com/api-platform/core/blob/main/src/Bridge/Symfony/Bundle/Resources/views/SwaggerUi/index.html.twig) and customize it.

## Compatibility Layer with Amazon API Gateway

[AWS API Gateway](https://aws.amazon.com/api-gateway/) supports OpenAPI partially, but it [requires some changes](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-known-issues.html).
API Platform provides a way to be compatible with Amazon API Gateway.

To enable API Gateway compatibility on your OpenAPI docs, add `api_gateway=true` as query parameter: `http://www.example.com/docs.json?api_gateway=true`.
The flag `--api-gateway` is also available through the command line.

## OAuth

If you implemented OAuth on your API, you should configure OpenApi's authorization using API Platform's configuration:

```yaml
api_platform:
    oauth:
        # To enable or disable oauth.
        enabled: false

        # The oauth client id.
        clientId: ''

        # The oauth client secret.
        clientSecret: ''

        # The oauth type.
        type: 'oauth2'

        # The oauth flow grant type.
        flow: 'application'

        # The oauth token url.
        tokenUrl: '/oauth/v2/token'

        # The oauth authentication url.
        authorizationUrl: '/oauth/v2/auth'

        # The oauth scopes.
        scopes: []
```

Note that `clientId` and `clientSecret` are being used by the SwaggerUI if enabled.

## Info Object

The [info object](https://swagger.io/specification/#info-object) provides metadata about the API like licensing information or a contact. You can specify this information using API Platform's configuration:

```yaml
api_platform:

    # The title of the API.
    title: 'API title'

    # The description of the API.
    description: 'API description'

    # The version of the API.
    version: '0.0.0'

    openapi:
        # The contact information for the exposed API.
        contact:
            # The identifying name of the contact person/organization.
            name:
            # The URL pointing to the contact information. MUST be in the format of a URL.
            url:
            # The email address of the contact person/organization. MUST be in the format of an email address.
            email:
        # A URL to the Terms of Service for the API. MUST be in the format of a URL.
        termsOfService:
        # The license information for the exposed API.
        license:
            # The license name used for the API.
            name:
            # URL to the license used for the API. MUST be in the format of a URL.
            url:
```