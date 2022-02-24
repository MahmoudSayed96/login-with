
# Login with Provider [Google, Facebook,...] in Laravel 8

This repo for explain how Authentication using Google, In this i am going to use Laravel Socialite Package.

[Laravel Socialite](https://github.com/laravel/socialite)

## STEP 1

Install package with composer.
```bash
composer require laravel/socialite
```
Include Package in Providers.
In `config/app.php` file add the below provider.
```php
Laravel\Socialite\SocialiteServiceProvider::class,
```
Scroll Down to aliases section and add below code.
```php
'Socialite' => Laravel\Socialite\Facades\Socialite::class,
```

## STEP 2
> Create model `Provider` and table for registeration data of social provider `-m` for migration.

```bash
php artisan make:model Provider -m
```

Update providers table
```php
Schema::create('providers', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('provider_id');
    $table->string('provider_name');
    $table->unsignedBigInteger('user_id');
    $table->timestamps();
});
```
Run this command for migrate table.
```php
php artisan migrate
```

## STEP 3
Create Controller for auth with provider.
```bash
php artisan make:controller AuthSocialiteController
```


## STEP 4
Add routes to `web.php`.

```php
// Auth with provider routes.
Route::get('auth/{provider}/redirect', 'AuthSocialiteController@redirectToProvider')->name('login_with_redirect');
Route::get('auth/{provider}/callback', 'AuthSocialiteController@handleProviderCallback')->name('login_with_callback');
```

## STEP 5
Add functions to controller.
```php
<?php

namespace App\Http\Controllers;

use App\Models\Provider;
use App\Models\User;
use Illuminate\Support\Facades\Auth;
use Laravel\Socialite\Facades\Socialite;

class AuthSocialiteController extends Controller
{
    public function redirectToProvider($provider)
    {
        return Socialite::driver($provider)->redirect();
    }

    public function handleProviderCallback($provider)
    {
        try {
            $user = Socialite::driver($provider)->user();
            // Check user if already logged with provider.
            $is_user_exists = Provider::where('user_id', $user->getId())->first();
            if (!$is_user_exists) {
                // Check if user already registered by email.
                $registeredUser = User::where('email', $user->getEmail())->first();
                if (!$registeredUser) {
                    // Create new.
                    $registeredUser = User::updateOrCreate([
                        'email' => $user->getEmail(),
                        'password' => bcrypt($user->getName() . '@' . $user->getId()),
                        'phone' => null,
                    ]);
                }
                // Create provider.
                Provider::create([
                    'provider_name' => $provider,
                    'provider_id' => $user->getId(),
                    'user_id' => $registeredUser->id,
                ]);
                Auth::loginUsingId($registeredUser->id);
            } else {
                Auth::loginUsingId($is_user_exists->id);
            }
            return redirect()->route('frontend.get_complete_profile');
        } catch (\Throwable $th) {
            throw $th;
        }
    }
}
```

## STEP 6
Create service account on google console for `client_id` and `client_secret`.
Add Provider Service in `config/services.php` File.
```php
'google' => [
    'client_id' => env('GOOGLE_CLIENT_ID'),
    'client_secret' => env('GOOGLE_CLIENT_SECRET'),
    'redirect' => env('GOOGLE_REDIRECT_URL'),
],
```

### Laravel Socialite: InvalidStateException
The Solution

Go to your www root, check the laravel file `config/session.php`
Check session Session Cookie Domain The default configuration is `'domain' => null`,
I made a change to `'domain' => 'mysite.com'`.
After `php artisan cache:clear` and `composer dump-autoload`, 
I can login with no issue from both `www.mysite.com` and `mysite.com`
Be sure to delete your cookies from browser when testing it after 
these modifications are done. Old cookies can still produce problems.
```
[Soluation](https://stackoverflow.com/questions/30660847/laravel-socialite-invalidstateexception)
