# Laravel Passport Multi-Auth

[![Latest Stable Version](https://poser.pugx.org/smartins/passport-multiauth/v/stable)](https://packagist.org/packages/smartins/passport-multiauth)
[![Latest Unstable Version](https://poser.pugx.org/smartins/passport-multiauth/v/unstable)](https://packagist.org/packages/smartins/passport-multiauth)
[![Build Status](https://travis-ci.org/sfelix-martins/passport-multiauth.svg?branch=master)](https://travis-ci.org/sfelix-martins/passport-multiauth)
[![Code Coverage](https://scrutinizer-ci.com/g/sfelix-martins/passport-multiauth/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/sfelix-martins/passport-multiauth/?branch=master)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/sfelix-martins/passport-multiauth/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/sfelix-martins/passport-multiauth/?branch=master)
[![StyleCI](https://styleci.io/repos/106218586/shield?branch=master)](https://styleci.io/repos/106218586)
[![License](https://poser.pugx.org/smartins/passport-multiauth/license)](https://packagist.org/packages/smartins/passport-multiauth)

Add multi-authentication support to [Laravel Passport](https://laravel.com/docs/5.5/passport)

## Upgrading from 1.0 to 2.0

- To upgrade from version 1.0 to 2.0 follow [this guide](https://github.com/sfelix-martins/passport-multiauth/blob/2.0/UPGRADE.md)

## Compatibility

| Laravel Passport |
|------------------|
| ^3.0             |
| ^4.0             |
| ^5.0             |
| ^6.0             |

## Installing and configuring

- Install using composer:

```sh
$ composer require smartins/passport-multiauth
```

- If you are using a Laravel version **less than 5.5** you **need to add** the provider on `config/app.php`:

```php
    'providers' => [
        // ...
        SMartins\PassportMultiauth\Providers\MultiauthServiceProvider::class,
    ],
```

- Migrate database to create `oauth_access_token_providers` table:

```sh
$ php artisan migrate
```

- Add new provider in `config/auth.php` using a model that extends of `Authenticatable` class and use `HasApiTokens` trait.

Example:

- Configure your model:

```php

use Laravel\Passport\HasApiTokens;
use Illuminate\Notifications\Notifiable;
use Illuminate\Foundation\Auth\User as Authenticatable;

class Admin extends Authenticatable
{
    use Notifiable, HasApiTokens;
}

```

- And your `config/auth.php` providers:

```php
    // ...

    'providers' => [
        'users' => [
            'driver' => 'eloquent',
            'model' => App\User::class,
        ],

        // ** New provider**
        'admins' => [
            'driver' => 'eloquent',
            'model' => App\Admin::class,
        ],
    ],

    // ...
```

- Add a new `guard` in `config/auth.php` guards array using driver `passport` and the provider added above:

```php
    // ...

    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'api' => [
            'driver' => 'passport',
            'provider' => 'users',
        ],

        // ** New guard **
        'admin' => [
            'driver' => 'passport',
            'provider' => 'admins',
        ],
    ],

    // ...
```

- Register the middleware `AddCustomProvider` to `$routeMiddleware` attributes on `app/Http/Kernel.php` file.

```php

class Kernel extends HttpKernel
{
    // ...

    /**
     * The application's route middleware.
     *
     * These middleware may be assigned to groups or used individually.
     *
     * @var array
     */
    protected $routeMiddleware = [
        // ...
        'oauth.providers' => \SMartins\PassportMultiauth\Http\Middleware\AddCustomProvider::class,
    ];

    // ...
}
```

- Replace the middleware `Authenticate` on `app/Http/Kernel` `$routeMiddleware` attribute.

```php

class Kernel extends HttpKernel
{
    // ...

    /**
     * The application's route middleware.
     *
     * These middleware may be assigned to groups or used individually.
     *
     * @var array
     */
    protected $routeMiddleware = [
        // 'auth' => \Illuminate\Auth\Middleware\Authenticate::class,
        'auth' => \SMartins\PassportMultiauth\Http\Middleware\MultiAuthenticate::class,
        'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
        'bindings' => \Illuminate\Routing\Middleware\SubstituteBindings::class,
        'can' => \Illuminate\Auth\Middleware\Authorize::class,
        'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
        'oauth.providers' => \SMartins\PassportMultiauth\Http\Middleware\AddCustomProvider::class,
        'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
    ];

    // ...
}
```

- Encapsulate the passport routes for access token with the registered middleware in `AuthServiceProvider`. This middleware will add the capability to `Passport` route `oauth/token` use the value of `provider` param on request:

```php
namespace App\Providers;

use Route;
use Laravel\Passport\Passport;
use Illuminate\Support\Facades\Gate;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    // ...

    /**
     * Register any authentication / authorization services.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();

        // Middleware `oauth.providers` middleware defined on $routeMiddleware above
        Route::group(['middleware' => 'oauth.providers'], function () {
            Passport::routes(function ($router) {
                return $router->forAccessTokens();
            });
        });
    }
    // ...
}
```

**Optional:** Publish migrations:

Just run the `vendor:publish` artisan command with package provider as parameter:

```sh
$ php artisan vendor:publish --provider="SMartins\PassportMultiauth\Providers\MultiauthServiceProvider"
```

## Usage

- Add the `provider` parameter in your request at `/oauth/token`:

```http
POST /oauth/token HTTP/1.1
Host: localhost
Accept: application/json, text/plain, */*
Content-Type: application/json;charset=UTF-8
Cache-Control: no-cache

{
    "username":"user@domain.com",
    "password":"password",
    "grant_type" : "password",
    "client_id": "client-id",
    "client_secret" : "client-secret",
    "provider" : "admins"
}
```

- You can pass your guards on `auth` middleware as you wish. Example:

```php
Route::group(['middleware' => ['api', 'auth:admin']], function () {
    Route::get('/admin', function ($request) {
        // Get the logged admin instance
        return $request->user(); // You can use too `$request->user('admin')` passing the guard.
    });
});

```

The  `api` guard use is equals the example with `admin`.

- You can pass many guards to `auth` middleware.

```php
Route::group(['middleware' => ['api', 'auth:admin,api']], function () {
    Route::get('/admin', function ($request) {
        // The instance of user authenticated (Admin or User in this case) will be returned
        return $request->user();
    });
});
```

You can use too the `Auth` facade:

```php
Auth::check();
Auth::user();
```

### Refreshing tokens

- Add the `provider` parameter in your request at `/oauth/token`:

```http
POST /oauth/token HTTP/1.1
Host: localhost
Accept: application/json, text/plain, */*
Content-Type: application/json;charset=UTF-8
Cache-Control: no-cache

{
    "grant_type" : "refresh_token",
    "client_id": "client-id",
    "client_secret" : "client-secret",
    "refresh_token" : "refresh-token",
    "provider" : "admins"
}
```

### Using scopes

- Just use the [`scope` and `scopes`](https://laravel.com/docs/5.5/passport#checking-scopes) middlewares from `Laravel\Passport`.

```php
protected $routeMiddleware = [
    'scopes' => \Laravel\Passport\Http\Middleware\CheckScopes::class,
    'scope' => \Laravel\Passport\Http\Middleware\CheckForAnyScope::class,
];
```

### Unit tests

Instead to use the `Laravel\Passport\Passport::actingAs()` method, use `SMartins\PassportMultiauth\PassportMultiauth::actingAs()`.
The difference is that the `actingAs` from this package get the guard based on `Authenticatable` instance passed on first parameter and authenticate this user using your guard. On authenticated request (Using `auth` middleware from package -  `SMartins\PassportMultiauth\Http\Middleware\MultiAuthenticate)` the guard is checked on `Request` to return the user or throws a `Unauthenticated` exception. E.g.:

```php
use App\User;
use Tests\TestCase;
use SMartins\PassportMultiauth\PassportMultiauth;

class AuthTest extends TestCase
{
    public function fooTest()
    {
        $user = factory(User::class)->create();

        PassportMultiauth::actingAs($user);

        $this->json('GET', 'api/user');
    }

    public function withScopesTest()
    {
        $user = factory(User::class)->create();

        PassportMultiauth::actingAs($user, ['see-balance']);

        $this->json('GET', 'api/balance');
    }
}
```

### Contributors

Based on [renanwilian](https://github.com/renanwilliam) responses to [Passport Multi-Auth issue](https://github.com/laravel/passport/issues/161).
