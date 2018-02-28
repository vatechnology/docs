---
title: Login
default: true
description: This tutorial demonstrates how to use the Auth0 PHP Laravel SDK to add authentication and authorization to your web app.
budicon: 448
github:
    path: 00-Starter-Seed
---
<%= include('../_includes/_getting_started', { library: 'Laravel', callback: 'http://localhost:3000/callback' }) %>

## Configure your application to use Auth0 

### Install the Auth0 plugin and its dependencies

${snippet(meta.snippets.dependencies)}

::: note
**[Composer](https://getcomposer.org/)** is a tool for dependency management in PHP. It allows you to declare the dependent libraries your project needs and it will install them in your project for you. See Composer's [getting started](https://getcomposer.org/doc/00-intro.md) doc for information on how to use it.
:::

### Enable it in Laravel

Add the following to the list of service providers, located in `config/app.php`

${snippet(meta.snippets.setup)}

Optionally, if you want to use the [facade](http://laravel.com/docs/facades) called `Auth0` you should also add an alias in the same file

```php
// config/app.php

'aliases' => array(
    // ...
    'Auth0' => Auth0\Login\Facade\Auth0::class
);
```

Finally, you will need to bind a class that provides the users (your app model user) each time a user is logged in or a JWT is decoded. You can use the `Auth0UserRepository` provided by this package or build your own (which should implement the `\Auth0\Login\Contract\Auth0UserRepository` interface, this is covered later).
For this you need to add to your `AppServiceProvider` the following line:


```php
// app/Providers/AppServiceProvider.php

public function register()
{

    $this->app->bind(
        \Auth0\Login\Contract\Auth0UserRepository::class, 
        \Auth0\Login\Repository\Auth0UserRepository::class
    );

}
```

### Configure It

To configure the plugin, you need to publish the plugin configuration and complete the file `config/laravel-auth0.php` using the information of your Auth0 account.

To publish the example configuration file use this command

```bash
php artisan vendor:publish
```

### Setup the Callback Action

The plugin works with the [Laravel authentication system](https://laravel.com/docs/5.3/authentication), but instead of using the `Auth::attempt` in a controller that handles a login form submit, you have to hook up the callback uri.

In other words, you need to select a uri (for example `/auth0/callback`) and configure it in your [Auth0 admin page](${manage_url}/#/applications) and also, add it as a route in `routes/web.php`:

```php
// routes/web.php

Route::get('/auth0/callback', '\Auth0\Login\Auth0Controller@callback');
```

## Trigger Authentication

You can trigger the login by calling `\App::make('auth0')->login();` in your controller, for example:

```php
// app/Http/Controllers/IndexController.php

class IndexController extends Controller {
    ...
    public function login()
    {
        return \App::make('auth0')->login(null, null, ['scope' => 'openid profile email'], 'code');
    }
}
```

Now, after user has logged in, you will be able to access to the logged user info with `Auth::user()`.

## Integrate with Laravel authentication system

The [Laravel authentication system](https://laravel.com/docs/5.5/authentication) needs a *User Object* given by a *User Provider*. With these two abstractions, the user entity can have any structure you like and can be stored anywhere. You configure the *User Provider* indirectly, by selecting a user provider in `app/config/auth.php`. The default provider is Eloquent, which persists the User model in a database using the ORM.

The plugin comes with an authentication driver called auth0. This driver defines a user structure that wraps the [Normalized User Profile](/user-profile) defined by Auth0, and it doesn't actually persist the object, it just stores it in the session for future calls.

This works fine for basic testing or if you don't really need to persist the user. At any point you can call `Auth::check()` to see if there is a user logged in and `Auth::user()` to get the wrapper with the user information.

Configure the `driver` in `/config/auth.php` to use `auth0`.

```php
// config/auth.php

// ...
'providers' => [
    'users' => [
        'driver' => 'auth0'
    ],
],
```

### Extend the `Auth0UserRepository` Class

There may be situations where you need to customize the `Auth0UserRepository` class. For example, you may want to use the default `User` model and store the user profile in your database. If you need a more advanced custom solution such as this, you can extend the `Auth0UserRepository` class with your own custom class.

::: note
This is an example, the custom class in this scenario will not work unless a database setup has been configured.
:::

```php
// app/Repository/MyCustomUserRepository.php

namespace App\Repository;

use Auth0\Login\Contract\Auth0UserRepository;

class MyCustomUserRepository implements Auth0UserRepository {

    /* This class is used on api authN to fetch the user based on the jwt.*/
    public function getUserByDecodedJWT($jwt) {
      /*
       * The `sub` claim in the token represents the subject of the token
       * and it is always the `user_id`
       */
      $jwt->user_id = $jwt->sub;

      return $this->upsertUser($jwt);
    }

    public function getUserByUserInfo($userInfo) {
      return $this->upsertUser($userInfo['profile']);
    }

    protected function upsertUser($profile) {

      // Note: Requires configured database access
      $user = User::where("auth0id", $profile->user_id)->first();

      if ($user === null) {
          // If not, create one
          $user = new User();
          $user->email = $profile->email; // you should ask for the email scope
          $user->auth0id = $profile->user_id;
          $user->name = $profile->name; // you should ask for the name scope
          $user->save();
      }

      return $user;
    }

    public function getUserByIdentifier($identifier) {
        //Get the user info of the user logged in (probably in session)
        $user = \App::make('auth0')->getUser();

        if ($user === null) return null;

        // build the user
        $user = $this->getUserByUserInfo($user);

        // it is not the same user as logged in, it is not valid
        if ($user && $user->auth0id == $identifier) {
            return $auth0User;
        }
    }

}
```

With your custom class in place, change the binding in the `AppServiceProvider`.

```php
// app/Providers/AppServiceProvider.php

// ...
public function register()
{

    $this->app->bind(
        \Auth0\Login\Contract\Auth0UserRepository::class, 
        \App\Repository\MyCustomUserRepository::class 
    );

}
```

### Use the Laravel authentication system

Now all your web routes will be secured by auth0.

For logging out your users, you can set up a route like this:

```php
Route::get('/logout', function() {
    Auth::logout();
    return Redirect::home();
});
```
