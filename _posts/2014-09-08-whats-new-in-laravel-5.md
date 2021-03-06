---
excerpt: Laravel 5 is coming in November, and some fundamental changes are coming for a minor version release. So what are they?
title: What’s new in Laravel 5
---
{% include alert.html type='info' message='This article is relevant to Laravel 5.0.x only.' %}

Laravel 5 is coming in November, and some fundamental changes. So, what are they?

## New directory structure
With Laravel 5 comes an overhaul of the directory structure of the starter app.
In version 5, the **app** directory now only contains application logic.
That’s pretty obvious when you think about it.
So if **app** only contains application logic what about, well, everything else?

Directories like **config**, **database**, **storage** and **tests** have been moved to the root.
The **lang** and **views** directories have been placed in a new **resources** directory, again in the root.
This leaves the directory structure in Laravel 5 looking like thus:

* app
  * Console
  * Http
    * Controllers
    * Filters
    * Requests
  * Providers
* bootstrap
* config
* database
  * migrations
  * seeds
* public
* resources
  * lang
  * views
* storage
  * cache
  * logs
  * meta
  * sessions
  * views
  * work

You can see that under the **app** directory there are three new sub-directories: **Console**, **Http**, and **Providers**.
In another makes-sense move, Laravel 5 has grouped logic into how the application is accessed.
So for example: you don’t use controllers when running Artisan console commands, so controllers have been placed in the **Http** sub-directory.
Similarly, you don’t need say, route filters when running console commands.

There’s also a new **Requests** sub-directory, and [requests](#requests) are something new to Laravel 5 but are going to save developers a _lot_ of time.
More on those below.

Laravel 5 will also make more use of service providers, hence the introduction of the **Providers** sub-directory.

## Name-spacing!
This is without a doubt my favourite enhancement in Laravel 5, and personally one I’m surprised has taken this long to come to Laravel: it’s name-spacing of the default app.

Previously, things like controllers and model classes were auto-loaded by Composer.
In version 5, Laravel’s gone down the [PSR-4](http://www.php-fig.org/psr/psr-4/) route of auto-loading classes and for this your application’s classes now needs a name-space.

Out of the box, this will simply be `App`. However, you can change this to something more unique with an Artisan command.
In a console, run:

```
$ php artisan app:name Acme
```

And this will update the name-space of _all_ your classes to be `Acme`.

**Note:** If you want to use a name-space more than one level deep, i.e. as per the common `Vendor\Project` convention, then use _two_ back-slashes, i.e.

```
$ php artisan app:name Acme\\AwesomeProject
```

## Requests
Laravel 5 introduces the notion of “requests”.
This is wrapping up logic that you would perform as part of a <abbr class="initialism" title="HyperText Transfer Protocol">HTTP</abbr> request, but are more than just a route filter.
A prime candidate: data validation.

Validation in Laravel is primarily performed using the in-built `Validator` class.
But even then, every developer has their own way of doing validation.

One method (that I have admittedly used) was to store validation rules in a `$rules` array in your model class, and for your model class to also have an `isValid()` method that performed the actual validation against its set attributes and return the result.
Purists would deride me (and others) for this approach, but it was simple and did the job for smaller apps.

Laravel has wrapped validation into request objects that can also contain authorisation.
Thinking in the case of registering, you would want to validate the data first.
A request object for that would look something like this:

```php
namespace App\Http\Requests\Auth;

use Illuminate\Foundation\Http\FormRequest;

class RegisterRequest extends FormRequest
{
    public function rules()
    {
        return [
            'email' => 'required|email|unique:users',
            'password' => 'required|confirmed|min:8',
        ];
    }

    public function authorize()
    {
        return true;
    }
}
```

There’s a `rules()` method that returns an array of rules you would before pass to `Validator::make()`,
and also an `authorize()` method where you would provide any user authorisation.
Usually you want all users to be able to register, so you just simply return `true`.

So how do you use this request class? This neatly leads me on to another feature…

## Method Parameter Injection
In Laravel, you could place type-hinted parameters to a controller’s `__construct` method,
and Laravel’s <abbr class="initialism" title="Inversion of Control">IoC</abbr> container would resolve it to that class
(or if it was an interface, its bound implementation in the container).
This lead to being able to do things like:

```php
class UserController extends BaseController
{
    public function __construct(UserRepositoryInterface $users)
    {
        $this->users = $users;
    }
}
```

Well, now developers can do the same with methods.
This means you can do something like:

```php
namespace App\Http\Controllers\Auth;

use Illuminate\Routing\Controller;
use Illuminate\Contracts\Auth\Authenticator;

use App\Http\Requests\Auth\RegisterRequest;

class AuthController extends Controller
{
    public function __construct(Authenticator $auth)
    {
        $this->auth = $auth;
    }

    public function postRegister(RegisterRequest $request)
    {
        // Registration form is valid; create user

        $this->auth->login($user);

        return redirect('/');
    }
}
```

If you look at the `postRegister()` method, you’ll see the `RegisterRequest` class is being injected.
Laravel will resolve this just like it resolves parameters specified in the constructor’s arguments list,
instantiate it, _and_ also automatically perform validation based on the request’s rules.
That means if validation fails, code in your `postRegister()` method is _never_ executed.
If name-spacing is my favourite part of Laravel 5, then this is a very close second!

With the above, you can flesh out the `postRegister()` method like this:

```php
public function postRegister(RegisterRequest $request)
{
    $user = User::create($request->all());

    $this->auth->login($user);

    return redirect('/');
}
```
This would create a new user.
The `all()` method may look familiar to you and that’s because it is: request classes in Laravel 5 extend `FormRequest`, which in turn extends `Request`.
That means you have access to form data and able to use methods like `all()` and `get()` to get at it.
There’s also a magic getter so you can access form data as properties, i.e. `$request->email`.
So in the above example, I’m just passing all form data to my user model’s `create()` method, which will mass-assign fillable keys.

## Other new stuff
There is a whole host of other new stuff in Laravel 5.
There are new Artisan commands, such as ones to generate boilerplate request classes.

### Auth controllers
Not only will Laravel 5 help set an ubiquitous approach to validation, but user authentication too.
There’s a handy new Artisan command that will generate you both an authentication _and_ password reminders controller!
This means you’ll seldom have to create a controller to handle logging in, registering, or resetting passwords again.

The command:

```
$ php artisan make:auth
```

The `AuthController` class earlier is an excerpt from what the command generates.

### Shortcuts
There are also shortcuts to `View::make()` and `Redirect`.
You’ve seen the redirect one above already: you can now just call `redirect()`.
Similarly, you can call `view()` as you would `View::make()`, passing the template name as the first parameter and view data as the second.

### Socialite
There’s also a new package called “Socialite” that will make working with third parties like Facebook and Twitter a breeze, and all via a common interface.
In fact, Laravel 5’s really going to town with interfaces, pushing the “program to interfaces and not implementations” paradigm.
So much so, there’s even a dedicated [Contracts](https://github.com/illuminate/contracts) repository housing all the interfaces Laravel’s [Illuminate](https://github.com/illuminate) framework uses under the hood.

This means you can pretty much see Laravel’s public <abbr class="initialism" title="Application Programming Interface">API</abbr> at a glance.
If you’re creating a new implementation for something (i.e. a database driver or custom authentication driver), then you can quickly look up the methods you need to implement so you don’t need to re-factor your app to use your new package.

I’m currently learning Objective-C, so the above makes sense when you think about a language like C (or one of its many derivative) where it defines public APIs in **.h** (header) files, and then implements it in the main **.c** files.

## Play with Laravel 5
Do you like to live on the edge?
Thinking about using Laravel for an upcoming project and want to get to grips with the next version?
If like me you couldn’t wait to get your hands on version 5 when learning what’s in store, you can check out the code today.

Assuming you have Composer installed on your machine (and you should), you can get Laravel 5 by running the following command:

```
$ composer create-project laravel/laravel laravel-5 dev-develop
```

This will install Laravel 5 to a new directory called **laravel-5**.
Simply change that if you want your directory called something else.

Happy tinkering! And do let me know your favourite features, or anything new that you’ve found in Laravel 5 yourself.
