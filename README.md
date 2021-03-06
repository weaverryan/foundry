# Foundry

[![CI Status](https://github.com/zenstruck/foundry/workflows/CI/badge.svg)](https://github.com/zenstruck/foundry/actions?query=workflow%3ACI)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/zenstruck/foundry/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/zenstruck/foundry/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/zenstruck/foundry/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/zenstruck/foundry/?branch=master)

A model factory library for creating expressive, auto-completable, on-demand test fixtures with Symfony and Doctrine.

Traditionally, data fixtures are defined in one or more files outside of your tests. When writing tests using these
fixtures, your fixtures are a sort of a *black box*. There is no clear connection between the fixtures and what you
are testing.

Foundry allows each individual test to fully follow the [AAA](https://www.thephilocoder.com/unit-testing-aaa-pattern/)
("Arrange", "Act", "Assert") testing pattern. You create your fixtures using "factories" at the beginning of each test.
You only create fixtures that are applicable for the test. Additionally, these fixtures are created with only the
attributes required for the test - attributes that are not applicable are filled with random data. The created fixture
objects are wrapped in a "proxy" that helps with pre and post assertions. 

Let's look at an example:

```php
public function test_can_post_a_comment(): void
{
    // 1. "Arrange"
    $post = PostFactory::new() // New Post factory
        ->published()          // Make the post in a "published" state
        ->create([             // Instantiate Post object and persist
            'slug' => 'post-a' // This test only requires the slug field - all other fields are random data
        ])
    ;
    
    // 1a. "Pre-Assertions"
    $this->assertCount(0, $post->getComments());

    // 2. "Act"
    $client = static::createClient();
    $client->request('GET', '/posts/post-a'); // Note the slug from the arrange step
    $client->submitForm('Add', [
        'comment[name]' => 'John',
        'comment[body]' => 'My comment',
    ]);

    // 3. "Assert"
    self::assertResponseRedirects('/posts/post-a');

    $this->assertCount(1, $post->getComments()); // $post is auto-refreshed before calling ->getComments()

    CommentFactory::repository()->assertExists([ // Doctrine repository wrapper with assertions
        'name' => 'John',
        'body' => 'My comment',
    ]);
}
```

## Documentation

1. [Installation](#installation)
2. [Test Traits](#test-traits)
3. [Faker](#faker)
4. [Sample Entities](#sample-entities)
5. [Anonymous Factories](#anonymous-factories)
    1. [Instantiate](#instantiate)
    2. [Persist](#persist)
    3. [Attributes](#attributes)
    4. [Doctrine Relationships](#doctrine-relationships)
    5. [Events](#events)
    6. [Instantiator](#instantiator)
    7. [Immutable](#immutable)
    8. [Object Proxy](#object-proxy)
    9. [Repository Proxy](#repository-proxy)
6. [Model Factories](#model-factories)
    1. [Generate](#generate)
    2. [Usage](#usage)
    3. [States](#states)
    4. [Initialize](#initialize)
7. [Stories](#stories)
    1. [Stories as Services](#stories-as-services)
8. [Seeding your Development Database](#seeding-your-development-database)
9. [Global Test State](#global-test-state)
10. [Full Default Bundle Configuration](#full-default-bundle-configuration)
11. [Using without the Bundle](#using-without-the-bundle)
12. [Performance Considerations](#performance-considerations)
    1. [DAMADoctrineTestBundle](#damadoctrinetestbundle)
    2. [Miscellaneous](#miscellaneous) 
13. [Credit](#credit)

### Installation

    $ composer require zenstruck/foundry --dev

To use the *Maker's*, ensure [Symfony MakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle/index.html)
is installed.

*If not using Symfony Flex, be sure to enable the bundle in your **test**/**dev** environments.*

### Test Traits

Add the `Factories` trait for tests using factories:

```php
use Zenstruck\Foundry\Test\Factories;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class MyTest extends WebTestCase // TestCase must be an instance of KernelTestCase
{
    use Factories;
    
    // ...
}
```

This library requires that your database be reset before each test. The packaged `ResetDatabase` trait handles this for
you. Before the first test, it drops (if exists) and creates the test database. Before each test, it resets the schema.

```php
use Zenstruck\Foundry\Test\Factories;
use Zenstruck\Foundry\Test\ResetDatabase;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class MyTest extends WebTestCase // TestCase must be an instance of KernelTestCase
{
    use ResetDatabase, Factories;
    
    // ...
}
```

**TIP**: Create a base TestCase for tests using factories to avoid adding the traits to every TestCase.

By default, `ResetDatabase` resets the default configured connection's database and default configured object manager's
schema. To customize the connection's and object manager's to be reset (or reset multiple connections/managers), set the
following environment variables:

```.env.test
FOUNDRY_RESET_CONNECTIONS=connection1,connection2
FOUNDRY_RESET_OBJECT_MANAGERS=manager1,manager2
```

### Faker

This library provides a wrapper for [fzaninotto/faker](https://github.com/fzaninotto/Faker) to help with generating
random data for your factories:

```php
use Zenstruck\Foundry\Factory;
use function Zenstruck\Foundry\faker;

Factory::faker()->name; // random name

// alternatively, use the helper function
faker()->email; // random email
```

**NOTE**: You can register your own `Faker\Generator`:

```yaml
# config/packages/dev/zenstruck_foundry.yaml and/or config/packages/test/zenstruck_foundry.yaml
zenstruck_foundry:
    faker:
        locale: fr_FR # set the locale
        # or
        service: my_faker # use your own instance of Faker\Generator for complete control
```

### Sample Entities

For the remainder of the documentation, the following sample entities will be used:

```php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity(repositoryClass="App\Repository\CategoryRepository")
 */
class Category
{
    /**
     * @ORM\Id()
     * @ORM\GeneratedValue()
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @ORM\Column(type="string", length=255)
     */
    private $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    // ... getters/setters
}
```

```php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity(repositoryClass="App\Repository\PostRepository")
 */
class Post
{
    /**
     * @ORM\Id()
     * @ORM\GeneratedValue()
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @ORM\Column(type="string", length=255)
     */
    private $title;

    /**
     * @ORM\Column(type="text", nullable=true)
     */
    private $body;
    
    /**
     * @ORM\Column(type="datetime")
     */
    private $createdAt;

    /**
     * @ORM\Column(type="datetime", nullable=true)
     */
    private $publishedAt;

    /**
     * @ORM\ManyToOne(targetEntity=Category::class)
     * @ORM\JoinColumn
     */
    private $category;

    public function __construct(string $title)
    {
        $this->title = $title;
        $this->createdAt = new \DateTime('now');
    }

    // ... getters/setters
}
```

### Anonymous Factories

Create a *Factory* for a class with the following:

```php
use App\Entity\Post;
use Zenstruck\Foundry\Factory;
use function Zenstruck\Foundry\factory;

new Factory(Post::class); // instance of Factory

factory(Post::class); // alternatively, use the helper function
```

#### Instantiate

Instantiate a factory object with the following:

```php
use App\Entity\Post;
use Zenstruck\Foundry\Factory;
use function Zenstruck\Foundry\faker;
use function Zenstruck\Foundry\instantiate;
use function Zenstruck\Foundry\instantiate_many;

(new Factory(Post::class))->withoutPersisting()->create(); // instance of Proxy (unpersisted) wrapping Post
(new Factory(Post::class))->withoutPersisting()->create()->object(); // instance of Post

// instance of Proxy wrapping Post with $title set to "Post A"
(new Factory(Post::class))->withoutPersisting()->create(['title' => 'Post A']); 

// array of 6 Proxy objects (unpersisted) wrapping Post objects with random titles
(new Factory(Post::class))->withoutPersisting()->createMany(6, function() {
    return ['title' => faker()->sentence];
});

// alternatively, use the helper functions
instantiate(Post::class, ['title' => 'Post A']);
instantiate_many(6, Post::class, function() {
    return ['title' => faker()->sentence];
});
```

#### Persist

Instantiate a factory object and persist to the database with the following (see the [Object Proxy](#object-proxy)
section for more information on the returned `Proxy` object):

```php
use App\Entity\Post;
use Zenstruck\Foundry\Factory;
use function Zenstruck\Foundry\faker;
use function Zenstruck\Foundry\create;
use function Zenstruck\Foundry\create_many;

// instance of Zenstruck\Foundry\Proxy wrapping Post with title "Post A"
(new Factory(Post::class))->create(['title' => 'Post A']);

// array of 6 Post Proxy objects with random titles
(new Factory(Post::class))->createMany(6, function() {
    return ['title' => faker()->sentence];
});

// alternatively, use the helper functions
create(Post::class, ['title' => 'Post A']);
create_many(6, Post::class, function() {
    return ['title' => faker()->sentence];
});
```

#### Attributes

The attributes used to instantiate the object can be added several ways. Attributes can be an *array*, or a *callable*
that returns an array. Using a *callable* helps with ensuring random data as the callable is run during instantiate.

```php
use App\Entity\Category;
use App\Entity\Post;
use Zenstruck\Foundry\Factory;
use function Zenstruck\Foundry\create;

$post = (new Factory(Post::class, ['title' => 'Post A']))
    ->withAttributes([
        'body' => 'Post A Body...',

        // can use snake case
        'published_at' => new \DateTime('now'), 

        // factories are automatically instantiated (and persisted if the outer Factory is persisted)
        'category' => new Factory(Category::class, ['name' => 'php']), 
    ])
    ->withAttributes([
        // can use kebab case
        'published-at' => new \DateTime('last week'),

        // Proxies are automatically converted to their wrapped object
        'category' => create(Category::class, ['name' => 'symfony']),
    ])
    ->withAttributes(function() { return ['createdAt' => Factory::faker()->dateTime]; })
    ->create(['title' => 'Different Title'])
;

$post->getTitle(); // "Different Title"
$post->getBody(); // "Post A Body..."
$post->getCategory()->getName(); // "symfony"
$post->getPublishedAt(); // \DateTime('last week')
$post->getCreatedAt(); // random \DateTime
```

#### Doctrine Relationships

Assuming your entites follow the
[best practices for Doctrine Relationships](https://symfony.com/doc/current/doctrine/associations.html) and you are
using the [default instantiator](#instantiator), Foundry *just works* with doctrine relationships:

```php
use function Zenstruck\Foundry\create;
use function Zenstruck\Foundry\factory;

// ManyToOne
create(Post::class, [
    'category' => $category, // $category is instance of Category
]);
create(Post::class, [
    // Proxy objects are converted to object before calling Post::setCategory()
    'category' => create(Category::class, ['name' => 'My Category']),
]);
create(Post::class, [
    // Factory objects are persisted before calling Post::setCategory()
    'category' => factory(Category::class, ['name' => 'My Category']),
]);

// OneToMany
create(Category::class, [
    'posts' => [
        $post, // $post is instance of Post, Category::addPost($post) will be called during instantiation

        // Proxy objects are converted to object before calling Category::addPost()
        create(Post::class, ['title' => 'Post B', 'body' => 'body']),

        // Factory objects are persisted before calling Category::addPost()
        factory(Post::class, ['title' => 'Post A', 'body' => 'body']),
    ],
]);

// ManyToMany
create(Post::class, [
    'tags' => [
        $tag, // $tag is instance of Tag, Post::addTag($tag) will be called during instantiation

        // Proxy objects are converted to object before calling Post::addTag()
        create(Tag::class, ['name' => 'My Tag']),

        // Factory objects are persisted before calling Post::addTag()
        factory(Tag::class, ['name' => 'My Tag']),
    ],
]);
```

#### Events

The following events can be added to factories. Multiple event callbacks can be added, they are run in the order
they were added.

```php
use Zenstruck\Foundry\Factory;
use Zenstruck\Foundry\Proxy;

(new Factory(Post::class))
    ->beforeInstantiate(function(array $attributes): array {
        // $attributes is what will be used to instantiate the object, manipulate as required
        $attributes['title'] = 'Different title';

        return $attributes; // must return the final $attributes
    })
    ->afterInstantiate(function(Post $object, array $attributes): void {
        // $object is the instantiated object
        // $attributes contains the attributes used to instantiate the object and any extras
    })
    ->afterPersist(function(Proxy $object, array $attributes) {
        /* @var Post $object */
        // this event is only called if the object was persisted
        // $proxy is a Proxy wrapping the persisted object
        // $attributes contains the attributes used to instantiate the object and any extras
    })

    // if the first argument is type-hinted as the object, it will be passed to the closure (and not the proxy)
    ->afterPersist(function(Post $object, array $attributes) {
        // this event is only called if the object was persisted
        // $object is the persisted Post object
        // $attributes contains the attributes used to instantiate the object and any extras
    })
    
    // multiple events are allowed
    ->beforeInstantiate(function($attributes) { return $attributes; })
    ->afterInstantiate(function() {})
    ->afterPersist(function() {})
;
```

#### Instantiator

By default, objects are instantiated with the object's constructor. Attributes that match constructor arguments are
used. Remaining attributes are set to the object using Symfony's PropertyAccess component (using setters). Any extra
attributes cause an exception to be thrown.

When using the default instantiator, there are two attribute key prefixes to change behavior:

```php
$post = (new Factory(Post::class))->create([
    'force:body' => 'some body', // "force set" the body property (even private/protected, does not use setter)
    'optional:extra' => 'value', // attributes prefixed with "optional:" do not cause an exception
]);
```

You can customize the instantiator several ways:

```php
use Zenstruck\Foundry\Instantiator;
use Zenstruck\Foundry\Factory;

// set the instantiator for the current factory
(new Factory(Post::class))
    // instantiate the object without calling the constructor
    ->instantiateWith((new Instantiator())->withoutConstructor())

    // extra attributes are ignored
    ->instantiateWith((new Instantiator())->allowExtraAttributes())

    // never use setters, always "force set" properties (even private/protected, does not use setter)
    ->instantiateWith((new Instantiator())->alwaysForceProperties())

    // can combine the different "modes"
    ->instantiateWith((new Instantiator())->withoutConstructor()->allowExtraAttributes()->alwaysForceProperties())

    // the instantiator is just a callable, you can provide your own
    ->instantiateWith(function(array $attibutes, string $class): object {
        return new Post(); // ... your own logic
    })
;
```

You can also customize the instantiator globally for all your factories (can still be overruled by factory instance
instantiators):

```yaml
# config/packages/dev/zenstruck_foundry.yaml and/or config/packages/test/zenstruck_foundry.yaml
zenstruck_foundry:
    instantiator:
        without_constructor: true # always instantiate objects without calling the constructor
        allow_extra_attributes: true # always ignore extra attributes
        always_force_properties: true # always "force set" properties
        # or
        service: my_instantiator # your own invokable service for complete control
```

#### Immutable

Factory objects are immutable:

```php
use App\Post;
use Zenstruck\Foundry\Factory;

$factory = new Factory(Post::class);
$factory->withAttributes([]); // new object
$factory->instantiateWith(function () {}); // new object
$factory->beforeInstantiate(function () {}); // new object
$factory->afterInstantiate(function () {}); // new object
$factory->afterPersist(function () {}); // new object
```

#### Object Proxy

Objects created by a factory are wrapped in a special [Proxy](src/Proxy.php) object. These objects help
with your "post-act" test assertions. Almost all calls to Proxy methods, first refresh the object from the database
(even if your entity manager has been cleared).

```php
use App\Entity\Post;
use Zenstruck\Foundry\Proxy;
use function Zenstruck\Foundry\create;

$post = create(Post::class, ['title' => 'My Title']); // instance of Zenstruck\Foundry\Proxy

// get the wrapped object
$post->object(); // instance of Post

// PHPUnit assertions
$post->assertPersisted();
$post->assertNotPersisted();

// delete from the database
$post->remove();

// call any Post methods - before calling, the object is auto-refreshed from the database
$post->getTitle(); // "My Title"

// set property and save to the database
$post->setTitle('New Title');
$post->save(); 

/**
 * CAVEAT - When calling multiple methods that change the object state, the previous state will be lost because
 * of auto-refreshing. Use "disableAutoRefresh()" or "withoutAutoRefresh()" to overcome this.
 */
$post->disableAutoRefresh();    // disable auto-refreshing
$post->refresh();               // manually refresh
$post->setTitle('New Title');   // won't be auto-refreshed
$post->setBody('New Body');     // won't be auto-refreshed
$post->save();                  // save changes (auto-refreshing re-enabled)

// alternatively, use "withAutoRefresh()" - auto-refreshing disabled before running callback and re-enabled after
$post->withoutAutoRefresh(function(Proxy $post) {
    /* @var Post $post */
    $post->setTitle('New Title');
    $post->setBody('New Body');
    $post->save();
});

// if the first argument is type-hinted as the wrapped object, it will be passed to the closure (and not the proxy)
$post->withoutAutoRefresh(function(Post $post) {
    $post->setTitle('New Title');
    $post->setBody('New Body');
})->save();

// set private/protected properties
$post->forceSet('createdAt', new \DateTime()); 
$post->forceSet('created_at', new \DateTime()); // can use snake case
$post->forceSet('created-at', new \DateTime()); // can use kebab case

/**
 * CAVEAT - When force setting multiple properties, the previous set's changes will be lost because
 * of auto-refreshing. Use "disableAutoRefresh()"/"withoutAutoRefresh()" (shown above) or "forceSetAll()"
 * to overcome this.
 */
$post->forceSetAll([
    'title' => 'Different title',
    'createdAt' => new \DateTime(),
]);

// get private/protected properties
$post->forceGet('createdAt');
$post->forceGet('created_at');
$post->forceGet('created-at');

$post->repository(); // instance of RepositoryProxy wrapping PostRepository
```

#### Repository Proxy

This library provides a Repository Proxy that wraps your object repositories to provide useful assertions and methods:

```php
use App\Entity\Post;
use function Zenstruck\Foundry\repository;

// instance of RepositoryProxy that wraps PostRepository
$repository = repository(Post::class);

// PHPUnit assertions
$repository->assertEmpty();
$repository->assertCount(3);
$repository->assertCountGreaterThan(3);
$repository->assertCountGreaterThanOrEqual(3);
$repository->assertCountLessThan(3);
$repository->assertCountLessThanOrEqual(3);
$repository->assertExists(['title' => 'My Title']);
$repository->assertNotExists(['title' => 'My Title']);

// helpful methods
$repository->getCount(); // number of rows in the database table
$repository->first(); // get the first object (wrapped in a object proxy)
$repository->truncate(); // delete all rows in the database table
$repository->random(); // get a random object
$repository->randomSet(5); // get 5 random objects
$repository->randomRange(0, 5); // get 0-5 random objects

// instance of ObjectRepository (all returned objects are proxied)
$repository->find(1);                               // Proxy|Post|null
$repository->find(['title' => 'My Title']);         // Proxy|Post|null
$repository->findOneBy(['title' => 'My Title']);    // Proxy|Post|null
$repository->findAll();                             // Proxy[]|Post[]
$repository->findBy(['title' => 'My Title']);       // Proxy[]|Post[]

// can call methods on the underlying repository (returned objects are proxied)
$repository->findOneByTitle('My Title'); // Proxy|Post|null
```

### Model Factories

You can create custom model factories to add IDE auto-completion and other useful features.

#### Generate

Create a model factory for one of your entities with the maker command:

    $ bin/console make:factory Post

**NOTES**:

1. Creates `PostFactory.php` in `src/Factory`, add `--test` flag to create in `tests/Factory`.
2. Calling `make:factory` without arguments displays a list of registered entities in your app to choose from.

Customize the generated model factory (if not using the maker command, this is what you will need to create manually):

```php
// src/Factory/PostFactory.php

namespace App\Factory;

use App\Entity\Post;
use App\Repository\PostRepository;
use Zenstruck\Foundry\RepositoryProxy;
use Zenstruck\Foundry\ModelFactory;
use Zenstruck\Foundry\Proxy;

/**
 * @method static Post|Proxy findOrCreate(array $attributes)
 * @method static Post|Proxy random()
 * @method static Post[]|Proxy[] randomSet(int $number)
 * @method static Post[]|Proxy[] randomRange(int $min, int $max)
 * @method static PostRepository|RepositoryProxy repository()
 * @method Post|Proxy create($attributes = [])
 * @method Post[]|Proxy[] createMany(int $number, $attributes = [])
 */
final class PostFactory extends ModelFactory
{
    protected function getDefaults(): array
    {
        return [
            'title' => self::faker()->unique()->sentence,
            'body' => self::faker()->sentence,
        ];
    }

    protected static function getClass(): string
    {
        return Post::class;
    }
}
```

**TIP**: It is best to have `getDefaults()` return the attributes to persist a valid object (all non-nullable fields).

#### Usage

Model factories extend `Zenstruck/Foundry/Factory` so all [methods and functionality](#anonymous-factories) are
available.

```php
use App\Factory\PostFactory;

$post = PostFactory::new()->create(); // Proxy with random data from `getDefaults()`

$post->getTitle(); // getTitle() can be autocompleted by your IDE!

PostFactory::new()->create(['title' => 'My Title']); // override defaults 
PostFactory::new(['title' => 'My Title'])->create(); // alternative to above

// find a persisted object for the given attributes, if not found, create with the attributes
PostFactory::findOrCreate(['title' => 'My Title']); // instance of Proxy|Post

// get a random object that has been persisted
PostFactory::random(); // instance of Proxy|Post

// get a random set of objects that have been persisted
PostFactory::randomSet(4); // array containing 4 instances of Proxy|Post's

// random range of persisted objects
PostFactory::randomRange(0, 5); // array containing 0-5 instances of Proxy|Post's

PostFactory::repository(); // Instance of RepositoryProxy wrapping PostRepository

// instantiate objects (without persisting)
PostFactory::new()->withoutPersisting()->create(); // instance of Proxy (unpersisted) wrapping Post object
PostFactory::new()->withoutPersisting()->create()->object(); // Post object

// instance of Proxy (unpersisted) wrapping Post object with $title = 'My Title'
PostFactory::new()->withoutPersisting()->create(['title' => 'My Title']);
```

#### States

You can add any methods you want to your model factories (ie static methods that create an object in a certain way) but
you can add "states":

```php
namespace App\Factory;

use App\Entity\Post;
use Zenstruck\Foundry\ModelFactory;
use function Zenstruck\Foundry\create;

final class PostFactory extends ModelFactory
{
    // ...

    public function published(): self
    {
        return $this->addState(['published_at' => self::faker()->dateTime]);
    }

    public function unpublished(): self
    {
        return $this->addState(['published_at' => null]);
    }

    public function withTags(string ...$tags): self
    {
        return $this->afterInstantiate(function (Post $post) use ($tags) {
            foreach ($tags as $tag) {
                $post->addTag(create(Tag::class, ['name' => $tag])->object());
            }
        });
    }

    public function withViewCount(int $count = null): self
    {
        return $this->addState(function () use ($count) {
            return ['view_count' => $count ?? self::faker()->numberBetween(0, 10000)];
        });
    }
}
```

You can use states to make your tests very explicit to improve readability:

```php
$post = PostFactory::new()->unpublished()->create();
$post = PostFactory::new()->withViewCount(3)->create();
$post = PostFactory::new()->withTags('dev', 'design')->create();

// combine multiple states
$post = PostFactory::new()
    ->withTags('dev')
    ->unpublished()
    ->create()
;

// states that don't require arguments can be added as strings to PostFactory::new()
$post = PostFactory::new('published', 'withViewCount')->create();
```

#### Initialize

You can override your model factory's `initialize()` method to add default state/logic:

```php
namespace App\Factory;

use App\Entity\Post;
use Zenstruck\Foundry\ModelFactory;
use function Zenstruck\Foundry\create;

final class PostFactory extends ModelFactory
{
    // ...

    protected function initialize(): self
    {
        return $this
            ->published() // published by default
            ->instantiateWith(function (array $attributes) {
                return new Post(); // custom instantiation for this factory
            })
            ->afterPersist(function () {}) // default event for this factory
        ; 
    }
}
```

### Stories

If you find your test's *arrange* step is getting complex (loading lots of fixtures) or duplicated, you can extract
this *state* into *Stories*. You can then load the Story at the beginning of these tests.

Create a story using the maker command:

    $ bin/console make:story Post

**NOTE**: Creates `PostStory.php` in `src/Story`, add `--test` flag to create in `tests/Story`.

Modify the *build* method to set the state for this story:

```php
// src/Story/PostStory.php

namespace App\Story;

use App\Factory\PostFactory;
use Zenstruck\Foundry\Story;

final class PostStory extends Story
{
    public function build(): void
    {
        // use "add" to have the object managed by the story and can be accessed in
        // tests and other stories via PostStory::postA()
        $this->add('postA', PostFactory::create(['title' => 'Post A']));

        // still persisted but not managed by the story
        PostFactory::create([
            'title' => 'Post D',
            'category' => CategoryStory::php(), // can use other stories
        ]);
    }
}
```

Use the new story in your tests:

```php
public function test_using_story(): void
{
    PostStory::load(); // loads the state defined in PostStory::build()

    PostStory::load(); // does nothing - already loaded

    PostStory::load()->get('postA'); // Proxy wrapping Post
    PostStory::load()->postA();      // alternative to above
    PostStory::postA();              // alternative to above

    PostStory::postA()->getTitle(); // "Post A"
}
```

**NOTE**: Story state and objects persisted by them are reset after each test.

#### Stories as Services

If you stories require dependencies, you can define them as a service:

```php
// src/Story/PostStory.php

namespace App\Story;

use App\Factory\PostFactory;
use App\Service\ServiceA;
use App\Service\ServiceB;
use Zenstruck\Foundry\Story;

final class PostStory extends Story
{
    private $serviceA;
    private $serviceB;

    public function __construct(ServiceA $serviceA, ServiceB $serviceB)
    {
        $this->serviceA = $serviceA;
        $this->serviceB = $serviceB;
    }

    public function build(): void
    {
        // can use $this->serviceA, $this->serviceB here to help build this story
    }
}
```

If using a standard Symfony Flex app, this will be autowired/autoconfigured but if not, register the service and tag
with `foundry.story`.

**NOTE:** The provided bundle is required for stories as services.

### Seeding your Development Database

Foundry works out of the box with [DoctrineFixturesBundle](https://symfony.com/doc/master/bundles/DoctrineFixturesBundle/index.html).
You can simply use your factory's and story's right within your fixture files:

```php
// src/DataFixtures/AppFixtures.php
namespace App\DataFixtures;

use App\Factory\CategoryFactory;
use App\Factory\PostFactory;
use App\Factory\TagFactory;
use App\Story\GlobalStory;
use Doctrine\Bundle\FixturesBundle\Fixture;
use Doctrine\Persistence\ObjectManager;

class AppFixtures extends Fixture
{
    public function load(ObjectManager $manager)
    {
        GlobalStory::load();

        // create 10 Category's
        CategoryFactory::new()->createMany(10);

        // create 20 Tag's
        TagFactory::new()->createMany(20);

        // create 50 Post's
        PostFactory::new()->createMany(50, function() {
            return [
                // each Post will have a random Category (created above)
                'category' => CategoryFactory::random(),

                // each Post will between 0 and 6 Tag's (created above)
                'tags' => TagFactory::randomRange(0, 6),
            ];
        });
    }
}
```

Run the [`doctrine:fixtures:load`](https://symfony.com/doc/master/bundles/DoctrineFixturesBundle/index.html#loading-fixtures)
as normal to seed your database.

### Global Test State

If you have an initial database state you want for all tests, you can set this in your `tests/bootstrap.php`:

```php
// tests/bootstrap.php
// ...

Zenstruck\Foundry\Test\TestState::addGlobalState(function () {
    CategoryFactory::create(['name' => 'php']);
    CategoryFactory::create(['name' => 'symfony']);
});
```

To avoid your boostrap file from becoming too complex, it is best to wrap your global state into a [Story](#stories):

```php
// tests/bootstrap.php
// ...

Zenstruck\Foundry\Test\TestState::addGlobalState(function () {
    GlobalStory::load();
});
```

**NOTE**: You can still access *Global State Stories* objects in your tests. They are still only loaded once.

### Full Default Bundle Configuration

```yaml
zenstruck_foundry:

    # Configure faker to be used by your factories.
    faker:

        # Change the default faker locale.
        locale:               null # Example: fr_FR

        # Customize the faker service.
        service:              null # Example: my_faker

    # Configure the default instantiator used by your factories.
    instantiator:

        # Whether or not to call an object's constructor during instantiation.
        without_constructor:  null

        # Whether or not to allow extra attributes.
        allow_extra_attributes: null

        # Whether or not to skip setters and force set object properties (public/private/protected) directly.
        always_force_properties: null

        # Customize the instantiator service.
        service:              null # Example: my_instantiator
```

### Using without the Bundle

The provided bundle is not strictly required to use Foundry. You can have all your factories, stories, and configuration
live in your `tests/` directory.

The best place to configure Foundry without the bundle is in your `tests/bootstrap.php` file:

```php
// tests/bootstrap.php
// ...

// required when not using the bundle so the test traits know not to look for it.
Zenstruck\Foundry\Test\TestState::withoutBundle();

// configure a default instantiator
Zenstruck\Foundry\Test\TestState::setInstantiator(
    (new Zenstruck\Foundry\Instantiator())
        ->withoutConstructor()
        ->allowExtraAttributes()
        ->alwaysForceProperties()
);

// configure a custom faker
Zenstruck\Foundry\Test\TestState::setFaker(Faker\Factory::create('fr_FR'));
```

### Performance Considerations

The following are possible options to improve the speed of your test suite.

#### DAMADoctrineTestBundle

This library integrates seamlessly with [DAMADoctrineTestBundle](https://github.com/dmaicher/doctrine-test-bundle) to
wrap each test in a transaction which dramatically reduces test time. This library's test suite runs 5x faster with
this bundle enabled.

Follow its documentation to install. Foundry's `ResetDatabase` trait detects when using the bundle and adjusts
accordingly. Your database is still reset before running your test suite but the schema isn't reset before each test
(just the first).

**NOTE**: If using [Global Test State](#global-test-state), it is persisted to the database (not in a transaction) before your
test suite is run. This could further improve test speed if you have a complex global state.

#### Miscellaneous

1. Disable debug mode when running tests. In your `.env.test` file, you can set `APP_DEBUG=0` to have your tests
run without debug mode. This can speed up your tests considerably. You will need to ensure you cache is cleared before
running the test suite. The best place to do this is in your `tests/bootstrap.php`:

    ```php
    // tests/bootstrap.php
    // ...
    if (false === (bool) $_SERVER['APP_DEBUG']) {
        // ensure fresh cache
        (new Symfony\Component\Filesystem\Filesystem())->remove(__DIR__.'/../var/cache/test');
    }
    ```

2. Reduce password encoder *work factor*. If you have a lot of tests that work with encoded passwords, this will cause
these tests to be unnecessarily slow. You can improve the speed by reducing the *work factor* of your encoder:

    ```yaml
    # config/packages/test/security.yaml
    encoders:
        # use your user class name here
        App\Entity\User:
            # This should be the same value as in config/packages/security.yaml
            algorithm: auto
            cost: 4 # Lowest possible value for bcrypt
            time_cost: 3 # Lowest possible value for argon
            memory_cost: 10 # Lowest possible value for argon
    ```

### Credit

The [AAA](https://www.thephilocoder.com/unit-testing-aaa-pattern/) style of testing was first introduced to me by
[Adam Wathan's](https://adamwathan.me/) excellent [Test Driven Laravel Course](https://course.testdrivenlaravel.com/).
The inspiration for this libraries API comes from [Laravel factories](https://laravel.com/docs/master/database-testing)
and [christophrumpel/laravel-factories-reloaded](https://github.com/christophrumpel/laravel-factories-reloaded).
