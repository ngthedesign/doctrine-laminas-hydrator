# doctrine-laminas-hydrator

[![Build Status](https://github.com/doctrine/doctrine-laminas-hydrator/workflows/Continuous%20Integration/badge.svg)](https://github.com/doctrine/doctrine-laminas-hydrator/actions?query=workflow%3A%22Continuous+Integration%22+branch%3A2.1.x)
[![Coverage Status](https://codecov.io/gh/doctrine/doctrine-laminas-hydrator/branch/4.0.x/graph/badge.svg)](https://codecov.io/gh/doctrine/dbal/branch/2.1.x)

This library provides Doctrine Hydrators for Laminas.

## Installation

Run the following to install this library:

```bash
$ composer require doctrine/doctrine-laminas-hydrator
```

## Usage

Hydrators convert an array of data to an object (this is called "hydrating") and
convert an object back to an array (this is called "extracting"). Hydrators are mainly used in the context of Forms,
with the binding functionality of Laminas, but can also be used in any hydrating/extracting context (for
instance, it can be used in RESTful context). If you are not really comfortable with hydrators, please first
read [Laminas hydrator's documentation](https://docs.laminas.dev/laminas-hydrator/).


### Basic usage

The library ships with a very powerful hydrator that allow almost any use-case.

#### Create a hydrator

To create a Doctrine Hydrator, you just need one thing: an object manager (also called Entity Manager in Doctrine ORM
or Document Manager in Doctrine ODM):

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($objectManager);
```

The hydrator constructor also allows a second parameter, `byValue`, which is true by default. We will come back later
about this distinction, but to be short, it allows the hydrator the change the way it gets/sets data by either
accessing the public API of your entity (getters/setters) or directly get/set data through reflection, hence bypassing
any of your custom logic.

#### Example 1: simple entity with no associations

Let's begin by a simple example:

```php
namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class City
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    protected ?int $id = null;

    #[ORM\Column(type: 'string', length: 48)]
    protected ?string $name = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function setName(string $name): void
    {
        $this->name = $name;
    }

    public function getName(): ?string
    {
        return $this->name;
    }
}
```

Now, let's use the Doctrine hydrator:

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$city = new City();
$data = [
    'name' => 'Paris',
];

$city = $hydrator->hydrate($data, $city);

echo $city->getName(); // prints "Paris"

$dataArray = $hydrator->extract($city);
echo $dataArray['name']; // prints "Paris"
```

As you can see from this example, in simple cases, the Doctrine Hydrator provides nearly no benefits over a
simpler hydrator like "ClassMethods". However, even in those cases, I suggest you to use it, as it performs automatic
conversions between types. For instance, it can convert timestamp to DateTime (which is the type used by Doctrine to
represent dates):

```php
namespace Application\Entity;

use DateTime;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Appointment
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    protected ?int $id = null;

    #[ORM\Column(type: 'datetime')]
    protected ?DateTime $time = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function setTime(DateTime $time): void
    {
        $this->time = $time;
    }

    public function getTime(): ?DateTime
    {
        return $this->time;
    }
}
```

Let's use the hydrator:

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$appointment = new Appointment();
$data = [
    'time' => '1357057334',
];

$appointment = $hydrator->hydrate($data, $appointment);

echo get_class($appointment->getTime()); // prints "DateTime"
```

As you can see, the hydrator automatically converted the timestamp to a DateTime object during the hydration, hence
allowing us to have a nice API in our entity with correct typehint.


#### Example 2: OneToOne/ManyToOne associations

Doctrine Hydrator is especially useful when dealing with associations (OneToOne, OneToMany, ManyToOne) and
integrates nicely with the Form/Fieldset logic ([learn more about this here](https://docs.laminas.dev/laminas-form/collections/)).

Let's take a simple example with a BlogPost and a User entity to illustrate OneToOne association:

```php
namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class User
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    protected ?int $id = null;

    #[ORM\Column(type: 'string', length: 48)]
    protected ?string $username = null;

    #[ORM\Column(type: 'string')]
    protected ?string $password = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function setUsername(string $username): void
    {
        $this->username = $username;
    }

    public function getUsername(): ?string
    {
        return $this->username;
    }

    public function setPassword(string $password): void
    {
        $this->password = $password;
    }

    public function getPassword(): ?string
    {
        return $this->password;
    }
}
```

And the BlogPost entity, with a ManyToOne association:

```php
namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class BlogPost
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    protected ?int $id = null;

    #[ORM\ManyToOne(targetEntity: 'Application\Entity\User')]
    protected ?User $user = null;

    #[ORM\Column(type: 'string')]
    protected ?string $title = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function setUser(User $user): void
    {
        $this->user = $user;
    }

    public function getUser(): ?User
    {
        return $this->user;
    }

    public function setTitle(string $title): void
    {
        $this->title = $title;
    }

    public function getTitle(): ?string
    {
        return $this->title;
    }
}
```

There are two use cases that can arise when using OneToOne association: the toOne entity (in this case, the User) may
already exist (which will often be the case with a User and BlogPost example), or it can be created. The
DoctrineHydrator natively supports both cases.

##### Existing entity in the association

When the association's entity already exists, all you need to do is simply give the identifier of the association:

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$blogPost = new BlogPost();
$data = [
    'title' => 'The best blog post in the world!',
    'user'  => [
        'id' => 2, // Written by user 2
    ],
];

$blogPost = $hydrator->hydrate($data, $blogPost);

echo $blogPost->getTitle(); // prints "The best blog post in the world!"
echo $blogPost->getUser()->getId(); // prints 2
```

**NOTE** : when using association whose primary key is not compound, you can rewrite the following more succinctly:

```php
$data = [
    'title' => 'The best blog post in the world!',
    'user'  => [
        'id' => 2, // Written by user 2
    ],
];
```

to:

```php
$data = [
    'title' => 'The best blog post in the world!',
    'user'  => 2,
];
```


##### Non-existing entity in the association

If the association's entity does not exist, you just need to give the object:

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$blogPost = new BlogPost();
$user = new User();
$user->setUsername('bakura');
$user->setPassword('p@$$w0rd');

$data = [
    'title' => 'The best blog post in the world!',
    'user'  => $user,
];

$blogPost = $hydrator->hydrate($data, $blogPost);

echo $blogPost->getTitle(); // prints "The best blog post in the world!"
echo $blogPost->getUser()->getId(); // prints 2
```

For this to work, you must also slightly change your mapping, so that Doctrine can persist new entities on
associations (note the cascade options on the ManyToOne association):

```php
namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class BlogPost
{
    /** .. */

    #[ORM\ManyToOne(targetEntity: 'Application\Entity\User', cascade: ['persist'])] 
    protected ?User $user = null;

    /** … */
}
```

It's also possible to use a nested fieldset for the User data.  The hydrator will
use the mapping data to determine the identifiers for the toOne relation and either
attempt to find the existing record or instanciate a new target instance which will
be hydrated before it is passed to the BlogPost entity.

**NOTE** : you're not really allowing users to be added via a blog post, are you?

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager, 'Application\Entity\BlogPost');
$blogPost = new BlogPost();

$data = [
    'title' => 'Art thou mad?',
    'user' => [
        'id' => '',
        'username' => 'willshakes',
        'password' => '2BorN0t2B',
    ],
];

$blogPost = $hydrator->hydrate($data, $blogPost);

echo $blogPost->getUser()->getUsername(); // prints willshakes
echo $blogPost->getUser()->getPassword(); // prints 2BorN0t2B
```


#### Example 3: OneToMany association

Doctrine Hydrator also handles OneToMany relationships (when use `Laminas\Form\Element\Collection` element). Please
refer to the official [Laminas documentation](https://docs.laminas.dev/laminas-form/collections/) to learn more about Collection.

> Note: internally, for a given collection, if an array contains identifiers, the hydrator automatically fetches the
objects through the Doctrine `find` function. However, this may cause problems if one of the values of the collection
is the empty string '' (as the ``find`` will most likely fail). In order to solve this problem, empty string identifiers
are simply ignored during the hydration phase. Therefore, if your database contains an empty string value as primary
key, the hydrator could not work correctly (the simplest way to avoid that is simply to not have an empty string primary
key, which should not happen if you use auto-increment primary keys, anyway).

Let's take again a simple example: a BlogPost and Tag entities.

```php
namespace Application\Entity;

use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class BlogPost
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    protected ?int $id = null;

    #[ORM\OneToMany(targetEntity: 'Application\Entity\Tag', mappedBy: 'blogPost')]
    protected Collection $tags;

    /**
     * Never forget to initialize your collections!
     */
    public function __construct()
    {
        $this->tags = new ArrayCollection();
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function addTags(Collection $tags): void
    {
        foreach ($tags as $tag) {
            $tag->setBlogPost($this);
            $this->tags->add($tag);
        }
    }

    public function removeTags(Collection $tags): void
    {
        foreach ($tags as $tag) {
            $tag->setBlogPost(null);
            $this->tags->removeElement($tag);
        }
    }

    public function getTags(): Collection
    {
        return $this->tags;
    }
}
```

And the Tag entity:

```php
namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Tag
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    protected ?int $id = null;

    #[ORM\ManyToOne(targetEntity: 'Application\Entity\BlogPost', inversedBy: 'tags')]
    protected ?BlogPost $blogPost = null;

    #[ORM\Column(type: 'string')]
    protected ?string $name = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    /**
     * Allow null to remove association
     */
    public function setBlogPost(?BlogPost $blogPost = null): void
    {
        $this->blogPost = $blogPost;
    }

    public function getBlogPost(): ?BlogPost
    {
        return $this->blogPost;
    }

    public function setName(string $name): void
    {
        $this->name = $name;
    }

    public function getName(): ?string
    {
        return $this->name;
    }
}
```

Please note some interesting things in BlogPost entity. We have defined two functions: addTags and removeTags. Those
functions must be always defined and are called automatically by Doctrine hydrator when dealing with collections.
You may think this is overkill, and ask why you cannot just define a `setTags` function to replace the old collection
by the new one:

```php
public function setTags(Collection $tags): void
{
    $this->tags = $tags;
}
```

But this is very bad, because Doctrine collections should not be swapped, mostly because collections are managed by
an ObjectManager, thus they must not be replaced by a new instance.

Once again, two cases may arise: the tags already exist or they do not.


##### Example 4: Embedded Entities

Doctrine provides so-called embeddables as a layer of abstraction which allow reusing partial object across entities. 
For example, one might have an entity `Address` which is not only used for a `Person`, but probably for an `Organisation`
as well. Let's have a look at the classes. First we have a `Tag` class, which will be our embeddable:

```php
namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * Address class for embedding in entities.
 */
#[ORM\Embeddable]
class Tag
{
    #[ORM\Column(type: 'string', nullable: true)]
    protected ?string $postalCode = null;

    #[ORM\Column(type: 'string', nullable: true)]
    protected ?string $city = null;

    public function getPostalCode(): ?string
    {
        return $this->postalCode;
    }

    public function setPostalCode(?string $postalCode): void
    {
        $this->postalCode = $postalCode;
    }

    public function getCity(): ?string
    {
        return $this->city;
    }
    
    public function setCity(?string $city): void
    {
        $this->city = $city;
    }
}
```

Then we have a corresponding `Person` entity, where the above embeddable is used:

```php
<?php

namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Person 
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    protected ?int $id = null;

    #[ORM\Column(type: 'string', nullable: true)]
    protected ?string $name = null;

    #[ORM\Embedded(class: 'Address')]
    protected Address $address;
    
    /**
     * Similar to collections you should initialize embeddables in the constructor!
     */
    public function __construct()
    {
        $this->address = new Address();
    }
    
    public function getId(): ?int
    {
        return $this->id;
    }
    
    public function getName(): ?string
    {
        return $this->name;
    }
    
    public function setName(?string $name): void
    {
        $this->name = $name;
    }

    public function getAddress(): Address
    {
        return $this->address;
    }
}
```

The hydrator provided by this module will require the data for the embeddable to be in a separate array as follows:

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$person = new Person();
$data = [
    'name' => 'Mr. Example',
    'address'  => [
        [
            'postalCode' => '48149',
            'city' => 'Münster',
        ],
    ],
];

$person = $hydrator->hydrate($data, $person);

echo $person->getAddress()->getPostalCode(); // prints "48149"
echo $person->getAddress()->getCity();       // prints "Münster"
```


##### Existing entity in the association

When the association's entity already exists, what you need to do is simply give the identifiers of the entities:

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$blogPost = new BlogPost();
$data = [
    'title' => 'The best blog post in the world!',
    'tags'  => [
        ['id' => 3], // add tag whose id is 3
        ['id' => 8], // also add tag whose id is 8
    ],
];

$blogPost = $hydrator->hydrate($data, $blogPost);

echo $blogPost->getTitle(); // prints "The best blog post in the world!"
echo count($blogPost->getTags()); // prints 2
```

**NOTE** : once again, this:

```php
$data = [
    'title' => 'The best blog post in the world!',
    'tags'  => [
        ['id' => 3], // add tag whose id is 3
        ['id' => 8], // also add tag whose id is 8
    ],
];
```

can be written:

```php
$data = [
    'title' => 'The best blog post in the world!',
    'tags'  => [3, 8],
];
```

##### Non-existing entity in the association

If the association's entity does not exist, you just need to give the object:

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$blogPost = new BlogPost();

$tags = [];

$tag1 = new Tag();
$tag1->setName('PHP');
$tags[] = $tag1;

$tag2 = new Tag();
$tag2->setName('STL');
$tags[] = $tag2;

$data = [
    'title' => 'The best blog post in the world!',
    'tags'  => $tags, // Note that you can mix integers and entities without any problem
];

$blogPost = $hydrator->hydrate($data, $blogPost);

echo $blogPost->getTitle(); // prints "The best blog post in the world!"
echo count($blogPost->getTags()); // prints 2
```

For this to work, you must also slightly change your mapping, so that Doctrine can persist new entities on
associations (note the cascade options on the OneToMany association):

```php
namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;
use Doctrine\Common\Collections\Collection;

#[ORM\Entity]
class BlogPost
{
    /** .. */

    #[ORM\OneToMany(targetEntity: 'Application\Entity\Tag', mappedBy: 'blogPost', cascade: ['persist'])]
    protected Collection $tags;

    /** … */
}
```

##### Handling of null values

When a null value is passed to a OneToOne or ManyToOne field, for example;

```php
$data = [
    'city' => null,
];
```

The hydrator will check whether the setCity() method on the Entity allows null values and act accordingly. The following describes the process that happens when a null value is received:

1. If the setCity() method DOES NOT allow null values i.e. `function setCity(City $city)`, the null is silently ignored and will not be hydrated.
2. If the setCity() method DOES allow null values i.e. `function setCity(City $city = null)`, the null value will be hydrated.

### Collections strategy

By default, every collections association has a special strategy attached to it that is called during the hydrating
and extracting phase. All those strategies extend from the class
`Doctrine\Laminas\Hydrator\Strategy\AbstractCollectionStrategy`.

The library provides four strategies out of the box:

1. `Doctrine\Laminas\Hydrator\Strategy\AllowRemoveByValue`: this is the default strategy, it removes old elements that are not in the new collection.
2. `Doctrine\Laminas\Hydrator\Strategy\AllowRemoveByReference`: this is the default strategy (if set to byReference), it removes old elements that are not in the new collection.
3. `Doctrine\Laminas\Hydrator\Strategy\DisallowRemoveByValue`: this strategy does not remove old elements even if they are not in the new collection.
4. `Doctrine\Laminas\Hydrator\Strategy\DisallowRemoveByReference`: this strategy does not remove old elements even if they are not in the new collection.

As a consequence, when using `AllowRemove*`, you need to define both adder (eg. addTags) and remover (eg. removeTags).
On the other hand, when using the `DisallowRemove*` strategy, you must always define at least the adder, but the remover
is optional (because elements are never removed).

The following table illustrates the difference between the two strategies

| Strategy | Initial collection | Submitted collection | Result |
| -------- | ------------------ | -------------------- | ------ |
| AllowRemove* | A, B  | B, C | B, C
| DisallowRemove* | A, B  | B, C | A, B, C

The difference between ByValue and ByReference is that when using strategies that end with ByReference, it won't use
the public API of your entity (adder and remover) - you don't even need to define them - it will directly add and
remove elements directly from the collection.


#### Changing the strategy

Changing the strategy for collections is plain easy.

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;
use Doctrine\Laminas\Hydrator\Strategy;

$hydrator = new DoctrineHydrator($entityManager);
$hydrator->addStrategy('tags', new Strategy\DisallowRemoveByValue());
```

Note that you can also add strategies to simple fields.


### By value and by reference

By default, Doctrine Hydrator works by value. This means that the hydrator will access and modify your properties
through the public API of your entities (that is to say, with getters and setters). However, you can override this
behaviour to work by reference (that is to say that the hydrator will access the properties through Reflection API,
and hence bypass any logic you may include in your setters/getters).

To change the behaviour, just give the second parameter of the constructor to false:

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($objectManager, false);
```

To illustrate the difference between, the two, let's do an extraction with the given entity:

```php
namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class SimpleEntity
{
    #[ORM\Column(type: 'string')]
    protected ?string $foo = null;

    public function getFoo(): void
    {
        die();
    }

    /** ... */
}
```

Let's now use the hydrator using the default method, by value:

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($objectManager);
$object   = new SimpleEntity();
$object->setFoo('bar');

$data = $hydrator->extract($object);

echo $data['foo']; // never executed, because the script was killed when getter was accessed
```

As we can see here, the hydrator used the public API (here getFoo) to retrieve the value.

However, if we use it by reference:

```php
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($objectManager, false);
$object   = new SimpleEntity();
$object->setFoo('bar');

$data = $hydrator->extract($object);

echo $data['foo']; // prints 'bar'
```

It now only prints "bar", which shows clearly that the getter has not been called.


### A complete example using Laminas\Form

Now that we understand how the hydrator works, let's see how it integrates into the Laminas' Form component.
We are going to use a simple example with, once again, a BlogPost and a Tag entities. We will see how we can create the
blog post, and being able to edit it.

#### The entities

First, let's define the (simplified) entities, beginning with the BlogPost entity:

```php
namespace Application\Entity;

use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class BlogPost
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    protected ?int $id = null;

    #[ORM\OneToMany(targetEntity: 'Application\Entity\Tag', mappedBy: 'blogPost', cascade: ['persist'])]
    protected Collection $tags;

    /**
     * Never forget to initialize your collections!
     */
    public function __construct()
    {
        $this->tags = new ArrayCollection();
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function addTags(Collection $tags): void
    {
        foreach ($tags as $tag) {
            $tag->setBlogPost($this);
            $this->tags->add($tag);
        }
    }

    public function removeTags(Collection $tags): void
    {
        foreach ($tags as $tag) {
            $tag->setBlogPost(null);
            $this->tags->removeElement($tag);
        }
    }

    public function getTags(): Collection
    {
        return $this->tags;
    }
}
```

And then the Tag entity:

```php
namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Tag
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    protected ?int $id = null;

    #[ORM\ManyToOne(targetEntity: 'Application\Entity\BlogPost', inversedBy: 'tags')]
    protected ?BlogPost $blogPost = null;

    #[ORM\Column(type: 'string')]
    protected ?string $name = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    /**
     * Allow null to remove association
     */
    public function setBlogPost(?BlogPost $blogPost = null): void
    {
        $this->blogPost = $blogPost;
    }

    public function getBlogPost(): ?BlogPost
    {
        return $this->blogPost;
    }

    public function setName(string $name): void
    {
        $this->name = $name;
    }

    public function getName(): ?string
    {
        return $this->name;
    }
}
```

#### The fieldsets

We now need to create two fieldsets that will map those entities. With Laminas it's a good practice to create
one fieldset per entity in order to reuse them across many forms.

Here is the fieldset for the Tag. Notice that in this example, I added a hidden input whose name is "id". This is
needed for editing. Most of the time, when you create the Blog Post for the first time, the tags do not exist.
Therefore, the id will be empty. However, when you edit the blog post, all the tags already exist in database (they
have been persisted and have an id), and hence the hidden "id" input will have a value. This allows you to modify a tag
name by modifying an existing Tag entity without creating a new tag (and removing the old one).

```php
namespace Application\Form;

use Application\Entity\Tag;
use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;
use Laminas\Form\Fieldset;
use Laminas\InputFilter\InputFilterProviderInterface;

class TagFieldset extends Fieldset implements InputFilterProviderInterface
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('tag');

        $this->setHydrator(new DoctrineHydrator($objectManager))
             ->setObject(new Tag());

        $this->add([
            'type' => 'Laminas\Form\Element\Hidden',
            'name' => 'id',
        ]);

        $this->add([
            'type'    => 'Laminas\Form\Element\Text',
            'name'    => 'name',
            'options' => [
                'label' => 'Tag',
            ],
        ]);
    }

    public function getInputFilterSpecification()
    {
        return [
            'id' => [
                'required' => false,
            ],
            'name' => [
                'required' => true,
            ],
        ];
    }
}
```

And the BlogPost fieldset:

```php
namespace Application\Form;

use Application\Entity\BlogPost;
use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;
use Laminas\Form\Fieldset;
use Laminas\InputFilter\InputFilterProviderInterface;

class BlogPostFieldset extends Fieldset implements InputFilterProviderInterface
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('blog-post');

        $this->setHydrator(new DoctrineHydrator($objectManager))
             ->setObject(new BlogPost());

        $this->add([
            'type' => 'Laminas\Form\Element\Text',
            'name' => 'title',
        ]);

        $tagFieldset = new TagFieldset($objectManager);
        $this->add([
            'type'    => 'Laminas\Form\Element\Collection',
            'name'    => 'tags',
            'options' => [
                'count'          => 2,
                'target_element' => $tagFieldset,
            ],
        ]);
    }

    public function getInputFilterSpecification()
    {
        return [
            'title' => [
                'required' => true,
            ],
        ];
    }
}
```

Plain and easy. The blog post is just a simple fieldset with an element type of ``Laminas\Form\Element\Collection``
that represents the ManyToOne association.

#### The form

Now that we have created our fieldset, we will create two forms: one form for creation and one form for updating.
The form's purpose is to be the glue between the fieldsets. In this simple example, both forms are exactly the same,
but in a real application, you may want to change this behaviour by changing the validation group (for instance, you
may want to disallow the user to modify the title of the blog post when updating).

Here is the create form:

```php
namespace Application\Form;

use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;
use Laminas\Form\Form;

class CreateBlogPostForm extends Form
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('create-blog-post-form');

        // The form will hydrate an object of type "BlogPost"
        $this->setHydrator(new DoctrineHydrator($objectManager));

        // Add the BlogPost fieldset, and set it as the base fieldset
        $blogPostFieldset = new BlogPostFieldset($objectManager);
        $blogPostFieldset->setUseAsBaseFieldset(true);
        $this->add($blogPostFieldset);

        // … add CSRF and submit elements …

        // Optionally set your validation group here
    }
}
```

And the update form:

```php
namespace Application\Form;

use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;
use Laminas\Form\Form;

class UpdateBlogPostForm extends Form
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('update-blog-post-form');

        // The form will hydrate an object of type "BlogPost"
        $this->setHydrator(new DoctrineHydrator($objectManager));

        // Add the BlogPost fieldset, and set it as the base fieldset
        $blogPostFieldset = new BlogPostFieldset($objectManager);
        $blogPostFieldset->setUseAsBaseFieldset(true);
        $this->add($blogPostFieldset);

        // … add CSRF and submit elements …

        // Optionally set your validation group here
    }
}
```

#### The controllers

We now have everything. Let's create the controllers.

##### Creation

If the createAction, we will create a new BlogPost and all the associated tags. As a consequence, the hidden ids
for the tags will by empty (because they have not been persisted yet).

Here is the action for create a new blog post:

```php
public function createAction()
{
    // Get your ObjectManager from the ServiceManager
    $objectManager = $this->getServiceLocator()->get('Doctrine\ORM\EntityManager');

    // Create the form and inject the ObjectManager
    $form = new CreateBlogPostForm($objectManager);

    // Create a new, empty entity and bind it to the form
    $blogPost = new BlogPost();
    $form->bind($blogPost);

    if ($this->request->isPost()) {
        $form->setData($this->request->getPost());

        if ($form->isValid()) {
            $objectManager->persist($blogPost);
            $objectManager->flush();
        }
    }

    return ['form' => $form];
}
```

The update form is similar, instead that we get the blog post from database instead of creating an empty one:

```php
public function editAction()
{
    // Get your ObjectManager from the ServiceManager
    $objectManager = $this->getServiceLocator()->get('Doctrine\ORM\EntityManager');

    // Create the form and inject the ObjectManager
    $form = new UpdateBlogPostForm($objectManager);

    // Fetch the existing BlogPost from storage and bind it to the form.
    // This will pre-fill form field values
    $blogPost = $this->userService->get($this->params('blogPost_id'));
    $form->bind($blogPost);

    if ($this->request->isPost()) {
        $form->setData($this->request->getPost());

        if ($form->isValid()) {
            // Save the changes
            $objectManager->flush();
        }
    }

    return ['form' => $form];
}
```

### Performance considerations

Although using the hydrator is like magical as it abstracts most of the tedious task, you have to be aware that it can
leads to performance issues in some situations. Please carefully read the following paragraphs in order to know how
to solve (and avoid!) them.

#### Unwanted side-effects

You have to be very careful when you are using Doctrine Hydrator with complex entities that contain a lot of
associations, as a lot of unnecessary calls to database can be made if you are not perfectly aware of what happen
under the hood. To explain this problem, let's have an example.

Imagine the following entity :


```php
namespace Application\Entity;

#[ORM\Entity]
class User
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    protected ?int $id = null;

    #[ORM\Column(type: 'string', length=48)]
    protected ?string $name = null;

    #[ORM\OneToOne(targetEntity: 'City')]
    protected ?City $city = null;

    // … getter and setters are defined …
}
```

This simple entity contains an id, a string property, and a OneToOne relationship. If you are using Laminas
forms the correct way, you will likely have a fieldset for every entity, so that you have a perfect mapping between
entities and fieldsets. Here are fieldsets for User and and City entities.

> If you are not comfortable with Fieldsets and how they should work, please refer to [this part of Laminas
documentation](https://docs.laminas.dev/laminas-form/collections/).

First the User fieldset :

```php
namespace Application\Form;

use Application\Entity\User;
use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;
use Laminas\Form\Fieldset;
use Laminas\InputFilter\InputFilterProviderInterface;

class UserFieldset extends Fieldset implements InputFilterProviderInterface
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('user');

        $this->setHydrator(new DoctrineHydrator($objectManager))
             ->setObject(new User());

        $this->add([
            'type'    => 'Laminas\Form\Element\Text',
            'name'    => 'name',
            'options' => [
                'label' => 'Your name',
            ],
            'attributes' => [
                'required' => 'required',
            ],
        ]);

        $cityFieldset = new CityFieldset($objectManager);
        $cityFieldset->setLabel('Your city');
        $cityFieldset->setName('city');
        $this->add($cityFieldset);
    }

    public function getInputFilterSpecification()
    {
        return [
            'name' => [
                'required' => true,
            ],
        ];
    }
}
```

And then the City fieldset :

```php
namespace Application\Form;

use Application\Entity\City;
use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;
use Laminas\Form\Fieldset;
use Laminas\InputFilter\InputFilterProviderInterface;

class CityFieldset extends Fieldset implements InputFilterProviderInterface
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('city');

        $this->setHydrator(new DoctrineHydrator($objectManager))
             ->setObject(new City());

        $this->add([
            'type'    => 'Laminas\Form\Element\Text',
            'name'    => 'name',
            'options' => [
                'label' => 'Name of your city',
            ],
            'attributes' => [
                'required' => 'required',
            ],
        ]);

        $this->add([
            'type'    => 'Laminas\Form\Element\Text',
            'name'    => 'postCode',
            'options' => [
                'label' => 'Postcode of your city',
            ],
            'attributes' => [
                'required' => 'required',
            ],
        ]);
    }

    public function getInputFilterSpecification()
    {
        return [
            'name' => [
                'required' => true,
            ],
            'postCode' => [
                'required' => true,
            ],
        ];
    }
}
```

Now, let's say that we have one form where a logged user can only change his name. This specific form does not allow
the user to change this city, and the fields of the city are not even rendered in the form. Naively, this form would
be like this :

```php
namespace Application\Form;

use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;
use Laminas\Form\Form;

class EditNameForm extends Form
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('edit-name-form');

        $this->setHydrator(new DoctrineHydrator($objectManager));

        // Add the user fieldset, and set it as the base fieldset
        $userFieldset = new UserFieldset($objectManager);
        $userFieldset->setName('user');
        $userFieldset->setUseAsBaseFieldset(true);
        $this->add($userFieldset);

        // … add CSRF and submit elements …

        // Set the validation group so that we don't care about city
        $this->setValidationGroup([
            'csrf', // assume we added a CSRF element
            'user' => [
                'name',
            ],
        ]);
    }
}
```

> Once again, if you are not familiar with the concepts here, please read the [official documentation about that](https://docs.laminas.dev/laminas-form/collections/).

Here, we create a simple form called "EditSimpleForm". Because we set the validation group, all the inputs related
to city (postCode and name of the city) won't be validated, which is exactly what we want. The action will look
something like this :

```php
public function editNameAction()
{
    // Get your ObjectManager from the ServiceManager
    $objectManager = $this->getServiceLocator()->get('Doctrine\ORM\EntityManager');

    // Create the form and inject the ObjectManager
    $form = new EditNameForm($objectManager);

    // Get the logged user (for more informations about userIdentity(), please read the Authentication doc)
    $loggedUser = $this->userIdentity();

    // We bind the logged user to the form, so that the name is pre-filled with previous data
    $form->bind($loggedUser);

    $request = $this->request;
    if ($request->isPost()) {
        // Set data from post
        $form->setData($request->getPost());

        if ($form->isValid()) {
            // You can now safely save $loggedUser
        }
    }
}
```

This looks good, doesn't it? However, if we check the queries that are made (for instance using the awesome
[Laminas\DeveloperTools module](https://github.com/laminas/laminas-developer-tools), we will see that a request is
made to fetch data for the City relationship of the user, and we hence have a completely useless database call,
as this information is not rendered by the form.

You could ask, "why?" Yes, we set the validation group, BUT the problem happens during the extracting phase. Here is
how it works : when an object is bound to the form, this latter iterates through all its fields, and tries to extract
the data from the object that is bound. In our example, here is how it works:

1. It first arrives to the UserFieldset. The input are "name" (which is string field), and a "city" which is another fieldset (in our User entity, this is a OneToOne relationship to another entity). The hydrator will extract both the name and the city (which will be a Doctrine 2 Proxy object).
2. Because the UserFieldset contains a reference to another Fieldset (in our case, a CityFieldset), it will, in turn, tries to extract the values of the City to populate the values of the CityFieldset. And here is the problem : City is a Proxy, and hence because the hydrator tries to extract its values (the name and postcode field), Doctrine will automatically fetch the object from the database in order to please the hydrator.

This is absolutely normal, this is how ZF forms work and what make them nearly magic, but in this specific case, it
can leads to disastrous consequences. When you have very complex entities with a lot of OneToMany collections, imagine
how many unnecessary calls can be made (actually, after discovering this problem, I've realized that my applications was
doing 10 unnecessary database calls).

In fact, the fix is ultra simple : if you don't need specific fieldsets in a form, remove them. Here is the fix
EditUserForm :

```php
namespace Application\Form;

use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Laminas\Hydrator\DoctrineObject as DoctrineHydrator;
use Laminas\Form\Form;

class EditNameForm extends Form
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('edit-name-form');

        $this->setHydrator(new DoctrineHydrator($objectManager));

        // Add the user fieldset, and set it as the base fieldset
        $userFieldset = new UserFieldset($objectManager);
        $userFieldset->setName('user');
        $userFieldset->setUseAsBaseFieldset(true);

        // We don't want City relationship, so remove it!!
        $userFieldset->remove('city');

        $this->add($userFieldset);

        // … add CSRF and submit elements …

        // We don't even need the validation group as the City fieldset does not
        // exist anymore
    }
}
```

And boom! Because the UserFieldset does not contain the CityFieldset relation anymore it won't be extracted.

As a rule of thumb, try to remove any unnecessary fieldset relationship, and always look at which database calls are made.
