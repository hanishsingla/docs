# Overriding Default Order

API Platform Core provides an easy way to override the default order of items in your collection.

By default, items in the collection are ordered in ascending (ASC) order by their resource identifier(s). If you want to
customize this order, you must add an `order` attribute on your ApiResource annotation:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(order: ["foo" => "ASC"])]
class Book
{
    // ...

    /**
     * ...
     */
    public $foo;
    
    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
App\Entity\Book:
    attributes:
        order:
            foo: ASC
```

[/codeSelector]

This `order` attribute is used as an array: the key defines the order field, the values defines the direction.
If you only specify the key, `ASC` direction will be used as default. For example, to order by `foo` & `bar`:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(order: ["foo", "bar"])]
class Book
{
    // ...

    /**
     * ...
     */
    public $foo;

    /**
     * ...
     */
    public $bar;
    
    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
App\Entity\Book:
    attributes:
        order: ['foo', 'bar']
```

[/codeSelector]

It's also possible to configure the default order on an association property:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(order: ["author.username"])]
class Book
{
    // ...

    /**
     * @var User
     */
    public $author;
    
    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
App\Entity\Book:
    attributes:
        order: ['author.username']
```

[/codeSelector]

Another possibility is to apply the default order for a specific collection operation, which will override the global default order configuration.

[codeSelector]

```php
#[ApiResource(
    collectionOperations: [
        "get",
        "get_desc_custom" => [
            "method" => "GET",
            "path" => "custom_collection_desc_foos",
            "order" => ["name" => "DESC"]
        ],
        "get_asc_custom" => [
            "method" => "GET",
            "path" => "custom_collection_asc_foos",
            "order" => ["name" => "ASC"]
        ],
    ]
)]
class Book
{
    // ...

    /**
     * @var string
     */
    public $name;
    
    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
App\Entity\Book:
    get: ~
    get_desc_custom:
        method: get
        path: custom_collection_desc_foos
        order:
            name: DESC
    get_asc_custom:
        method: get
        path: custom_collection_asc_foos
        order:
            name: ASC
```

[/codeSelector]
