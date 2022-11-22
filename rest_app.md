# Create a REST template for Controller using CakePHP (4.x)

Today APIs are everywhere and most of IT projects use multiple layers of logic.
In base of [CakePHP documentation](https://cakephp.org/), I will show you how to easily create RESTFUL routes & controllers.

## Initialize CakePHP project

First, you have to create & initialize a cakephp project using this command :
``` Shell
composer create-project --prefer-dist cakephp/app rest_app
```

![](/rest_01.png "rest_01")


Once the project is initialized, the first thing to do is create a database.
For the example I create a MYSQL user `rest_api` with `rest_api` as password & database name.

Open the `config/app_local.php` file and go to `Datasources` variable definition.
Replace the `default` definition by :

``` PHP
...
'default' => [
    'host' => 'localhost',
    'username' => 'rest_api',
    'password' => 'rest_api',
    'database' => 'rest_api'
],
'test' => [
    'host' => 'localhost',
    'username' => 'rest_api',
    'password' => 'rest_api',
    'database' => 'rest_api_test'
],
...
```

The project can now interact with database :

![](/rest_02.png "rest_02")

## Database Population

For the example we will create 2 tables (on the 2 databases) :

``` SQL
CREATE TABLE `rest_api`.`authors` (
    `id` INT NOT NULL AUTO_INCREMENT,
    `lastname` VARCHAR(255) NOT NULL ,
    `firstname` VARCHAR(255) NOT NULL ,
    PRIMARY KEY (`id`)
);

CREATE TABLE `rest_api`.`posts` (
    `id` INT NOT NULL AUTO_INCREMENT ,
    `text` VARCHAR(1023) NOT NULL ,
    `author_id` INT NOT NULL ,
    PRIMARY KEY (`id`)
);
```

We will also insert some values (only on 'default' database) :

``` SQL
INSERT INTO `authors` (`lastname`, `firstname`) VALUES ('Doe', 'John');
INSERT INTO `authors` (`lastname`, `firstname`) VALUES ('Foo', 'James');

INSERT INTO `posts`(`text`, `author_id`) VALUES ('This is the first post', 1);
INSERT INTO `posts`(`text`, `author_id`) VALUES ('This is the second post', 2);
INSERT INTO `posts`(`text`, `author_id`) VALUES ('This is the third post', 1);
```

## Models generation using `bake` command

You can **easily** and **rapidly** create models from your tables using the `bake` command.

Let's go for the `authors` table

For windows users :

``` Shell
"./bin/cake" bake model authors
```

For unix users :

``` Shell
./bin/cake bake model authors
```

It will create all classes defining the `Author` model.

Now we can do this for the `posts` tables

For windows users :

``` Shell
"./bin/cake" bake model posts
```

For unix users :

``` Shell
./bin/cake bake model posts
```

This time CakePHP will detect the association between with the `authors` table via the `author_id` field.

## Adapt routes for JSON extensions

Without modification you can handle JSON response ***within views***. But you can do this without views ***by adapting*** `config/routes.php` like this :

``` PHP
use Cake\Routing\Route\DashedRoute;
use Cake\Routing\RouteBuilder;

return static function (RouteBuilder $routes) {
    $routes->setRouteClass(DashedRoute::class);

    $routes->scope('/', function (RouteBuilder $routes) {
        $routes->connect('/', ['controller' => 'Pages', 'action' => 'display', 'home']);

        $routes->setExtensions(['json']);
        $routes->resources('{controller}');
    });
};
```

We let the home page but **all other pages** will have a ***JSON response***.

You can also add an scope for JSON responses but for **this example** we make a simple configuration.

## Disable CSRF protection for APIs

```PHP
declare(strict_types=1);

class Application extends BaseApplication
{
    ...
    public function middleware(MiddlewareQueue $middlewareQueue): MiddlewareQueue
    {
        $middlewareQueue
            ->add(new ErrorHandlerMiddleware(Configure::read('Error')))
            ->add(new AssetMiddleware([
                'cacheTime' => Configure::read('Asset.cacheTime'),
            ]))
            ->add(new RoutingMiddleware($this))
            ->add(new BodyParserMiddleware())
            ->add((new CsrfProtectionMiddleware([
                'httponly' => true,
            ]))->skipCheckCallback(function() {
                    return true;
                })
            );
        return $middlewareQueue;
    }
    ...
}

```

## Create the `RestController`

We will code ONE TIME the rest logic within the new `src/Controller/RestController.php`. Here is the base code for this controller :

``` PHP
declare(strict_types=1);
namespace App\Controller;

class RestController extends AppController {
    public $modelName = '';
    public $uniqueEntityName = '';
}

```

Usualy controllers extends the `AppController` class and each method is attached to a view.

This `RestController` class will replace `AppController` for the others controllers.

## Create AuthorsController

Once the base controller created we can create the specific ones.

Here is the `src/Controller/AuthorsController.php`

``` PHP
declare(strict_types=1);
namespace App\Controller;

class AuthorsController extends RestController {
    public function initialize(): void
    {
        parent::initialize();
        $this->$modelName = $this->getName();
        $this->$uniqueEntityName = 'Author';
    }
}
```

## Define actions into RestController

Based on [CakePHP documentation](https://cakephp.org/) here is the table according to HTTP Request Method (sensitive) :

| HTTP format | Controller action invoked | URL |
| - | - | - |
| GET | *index()* | `<YOUR_URL>`/`<YOUR_CONTROLLER>`.json |
| GET | *view(id)* | `<YOUR_URL>`/`<YOUR_CONTROLLER>`.id.json |
| POST | *add()* | `<YOUR_URL>`/`<YOUR_CONTROLLER>`.json |
| PUT | *edit(id)* | `<YOUR_URL>`/`<YOUR_CONTROLLER>`/id.json |
| PATCH | *edit(id)* | `<YOUR_URL>`/`<YOUR_CONTROLLER>`/id.json |
| DELETE | *delete(id)* | `<YOUR_URL>`/`<YOUR_CONTROLLER>`/id.json |

### GET (all) function

The `index` function will return all records of a table.

Here is an example :
``` PHP
class RestController extends AppController {
    public $modelName = '';
    public $uniqueEntityName = '';

    public function index() {
        $data = $this->{$this->modelName}->find()->all();
        $this->set($this->modelName, $data);
        $this->viewBuilder()
            ->setOption('serialize', [$this->modelName])
            ->setOption('jsonOptions', JSON_FORCE_OBJECT);
    }
}
```
The result will be for the url `<YOUR_URL>/authors.json`:
``` JSON
{
    "Authors": {
        "0": {
            "id": 1,
            "lastname": "Doe",
            "firstname": "John"
        },
        "1": {
            "id": 2,
            "lastname": "Foo",
            "firstname": "James"
        }
    }
}
```

### GET (by id) function

The `view` function will return the record according to the id passed into the function.

Here is an example :
```PHP
class RestController extends AppController {
    public $modelName = '';
    public $uniqueEntityName = '';

    ...

    public function view($id) {
        $data = $this->{$this->modelName}->get($id);
        $this->set($this->uniqueEntityName, $data);
        $this->viewBuilder()
            ->setOption('serialize', [$this->uniqueEntityName])
            ->setOption('jsonOptions', JSON_FORCE_OBJECT);
    }
}
```

The result will be for the url `<YOUR_URL>/authors/1.json`:
``` JSON
{
    "Author": {
        "id": 1,
        "lastname": "Doe",
        "firstname": "John"
    }
}
```

### POST function

The `add` function will add a new record into the database via the `POST` HTTP method.

Here is an example :

```PHP
class RestController extends AppController {
    public $modelName = '';
    public $uniqueEntityName = '';

    ...

    public function add() {
        $data = $this->{$this->modelName}->newEntity($this->request->getData());
        if ($this->{$this->modelName}->save($data)) {
            $message = 'Saved';
        } else {
            $message = 'Error';
        }
        $this->set([
            'message' => $message,
            $this->uniqueEntityName => $data,
        ]);
        $this->viewBuilder()->setOption('serialize', [$this->uniqueEntityName, 'message']);
    }
}
```

If we send those data :

``` JSON
{
    "lastname": "Smith",
    "firstname": "Tom"
}
```

The result will be :

``` JSON
{
    "Author": {
        "lastname": "Smith",
        "firstname": "Tom",
        "id": 3
    },
    "message": "Saved"
}
```

If the are error(s) the response will be :

``` JSON
{
    "Author": [],
    "message": "Error"
}
```

### PUT function

The `edit` function will edit a record into the database via the `PUT` OR `PATCH` HTTP method.

Here is an example :

```PHP
class RestController extends AppController {
    public $modelName = '';
    public $uniqueEntityName = '';

    ...

    public function edit($id) {
        $savedData = null;
        $message = '';
        try {
            $data = $this->{$this->modelName}->get($id);
        }
        catch (\Exception $exception) {
            $savedData = [];
            $message = 'Error';
        }
        if ($this->request->is(['post', 'put', 'patch'])) {
            $savedData = $this->{$this->modelName}->patchEntity($data, $this->request->getData());
            if ($this->{$this->modelName}->save($savedData)) {
                $message = 'Saved';
            }
        }
        $this->set([
            'message' => $message,
            $this->uniqueEntityName => $savedData,
        ]);
        $this->viewBuilder()->setOption('serialize', [$this->uniqueEntityName, 'message']);
    }
}
```

If we send those data :

``` JSON
{
    "lastname": "Smith",
    "firstname": "Thomas"
}
```

The result will be :

``` JSON
{
    "Author": {
        "id": 3,
        "lastname": "Smith",
        "firstname": "Thomas"
    },
    "message": "Saved"
}
```

### DELETE function

The `delete` function will remove a record from the database via the `DELETE` HTTP method.

Here is an example :

```PHP
class RestController extends AppController {
    public $modelName = '';
    public $uniqueEntityName = '';

    ...

    public function delete($id) {
        $data = null;
        $message = '';
        try {
            $data = $this->{$this->modelName}->get($id);
        }
        catch (\Exception $exception) {
            $message = 'Error';
        }
        if ($data != null) {
            $this->{$this->modelName}->delete($data);
            $message = 'Deleted';
        }
        $this->set('message', $message);
        $this->viewBuilder()->setOption('serialize', ['message']);
    }
}
```

If the record exists the result will be :

``` JSON
{
    "message": "Deleted"
}
```

If the record doesn't exist the result will be :

``` JSON
{
    "message": "Error"
}
```

## Create the PostsController

Once the `RestController` completed you can create the new controller for `Post` model into a file called `src/Controller/PostsController.php`.

``` PHP
declare(strict_types=1);

namespace App\Controller;

class PostsController extends RestController
{
    public function initialize(): void
    {
        parent::initialize();
        $this->modelName = $this->getName();
        $this->uniqueEntityName = 'Post';
    }
}

```

Each action defined into `RestController` is aotomaticaly available for this new controller !

Let's check with get all posts by url `<YOUR_URL>/posts.json` :

``` JSON
{
    "Posts": {
        "0": {
            "id": 1,
            "text": "This is the first post",
            "author_id": 1
        },
        "1": {
            "id": 2,
            "text": "This is the second post",
            "author_id": 2
        },
        "2": {
            "id": 3,
            "text": "This is the third post",
            "author_id": 1
        }
    }
}
```

By a miracle it's working !

But you can create your own logic for a specific action.

If you want to get Authors data for this function you can do like this :

``` PHP
declare(strict_types=1);

namespace App\Controller;

class PostsController extends RestController
{
    ...

    public function index() {
        $data = $this->Posts->find()->contain('Authors');
        $this->set($this->modelName, $data);
        $this->viewBuilder()
            ->setOption('serialize', [$this->modelName])
            ->setOption('jsonOptions', JSON_FORCE_OBJECT);
    }
}
```

And the result will be :

```JSON
{
    "Posts": {
        "0": {
            "id": 1,
            "text": "This is the first post",
            "author_id": 1,
            "author": {
                "id": 1,
                "lastname": "Doe",
                "firstname": "John"
            }
        },
        "1": {
            "id": 3,
            "text": "This is the third post",
            "author_id": 1,
            "author": {
                "id": 1,
                "lastname": "Doe",
                "firstname": "John"
            }
        },
        "2": {
            "id": 2,
            "text": "This is the second post",
            "author_id": 2,
            "author": {
                "id": 2,
                "lastname": "Foo",
                "firstname": "James"
            }
        }
    }
}
```

## Conclusion

It was easy to create a maintenable & reliable logic for REST handling and you can create or alter APIs controller rapidly.
