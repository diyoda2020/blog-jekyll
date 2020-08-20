---
layout: post
title:  "Tutorial Backend Laravel 7 Auth API Menggunakan Passport"
date:   2020-08-18 09:00:04 +0700
categories: Laravel Auth
youtubeId: Bis5AykCH4M
---
Sebelum kita menuju inti dari tutorial ini, kita lakukan instalasi laravel di lokal environment
komputer kita, dengan perintah sbb :

{% highlight php %}

composer create-project --prefer-dist laravel/laravel tutorialAuth

{% endhighlight %}

Link : [Dokumentasi instalasi Laravel](https://laravel.com/docs/7.x/installation){:target="_blank"}

Setelah instalasi laravel selesai kita lakukan, kita lakukan instalasi passport, dengan perintah sbb :

{% highlight php %}

composer require laravel/passport

{% endhighlight %}

Link : [Dokumentasi Passport Laravel](https://laravel.com/docs/7.x/passport){:target="_blank"}

Disini kita menggunakan metode Password Grant Token, jadi untuk mendapatkan token dari server, user harus menggunakan password.

sebelum melakukan migrate untuk membuat database, kita setting dahulu file .ENV, sesuai dengan 
database yang kita buat, disini saya beri contoh untuk database menggunakan mysql :

{% highlight php %}

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravelAuth
DB_USERNAME=root
DB_PASSWORD=password

{% endhighlight %}

Untuk mempermudah melakukan pengetesan, kita buat dahulu seed untuk user sebanyak 50 user, dengan merubah file DatabaseSeeder.php menjadi :

{% highlight php %}

<?php

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Seed the application's database.
     *
     * @return void
     */
    public function run()
    {
        //$this->call(UserSeeder::class);
        factory('App\User',50)->create();
    }
}

{% endhighlight %}


setelah itu kita lakukan perintah

{% highlight php %}

php artisan migrate --seed

{% endhighlight %}

kemudian kita buat secret key untuk password grand client, sesuai dengan metode passport yang akan kita pakai dengan perintah

{% highlight php %}

php artisan passport:install

atau

php artisan passport:client --password

{% endhighlight %}

Setelah itu, sesuai petunjuk dari dokumentasi laravel passport kita lakukan modifikasi pada 
Model User dengan menambahkan trait HasApiTokens menjadi :

{% highlight php %}

<?php

namespace App;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

// Tambahan trait dari Passport Package
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use Notifiable, HasApiTokens;

{% endhighlight %}

Untuk route dari Passport kita harus masukkan di file AuthServiceProvider.php pada function boot, begitu juga dengan konfigurasi dari token yang dihasilkan seperti masa berlaku token berapa lama dan lain sebagainya, berikut, saya beri contoh sesuai dokumentasi passport :

{% highlight php %}

<?php

namespace App\Providers;

use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Gate;

use Laravel\Passport\Passport;
use Carbon\Carbon;

class AuthServiceProvider extends ServiceProvider
{
    protected $policies = [
        // 'App\Model' => 'App\Policies\ModelPolicy',
    ];

    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();
        Passport::tokensExpireIn(Carbon::now()->addMinutes(5));
        Passport::refreshTokensExpireIn(Carbon::now()->addMinutes(720));
    }
}

{% endhighlight %}

Lalu kita modifikasi api driver dari token menjadi passport pada file config/auth.php, sebagaimana contoh berikut :

{% highlight php %}

'api' =>  [
            // 'driver' => 'token',
            'driver' => 'passport',
            'provider' => 'users',
            'hash' => false,
          ],

{% endhighlight %}

untuk emndapatkan token pertama kali dan token setelah login, kita buat trait untuk menampung function seperti yang di contohkan pada dokumentasi Passport, kita beri nama trait tersebut PassportTokenTraits, berikut isi file traitnya :


{% highlight php %}

<?php

namespace App\CustomTraits;

use Laravel\Passport\Client as PClient;
use GuzzleHttp\Client;

trait PassportTokenTrait {

    /**
     * First Token Function.
     */
    public function getFirstTokenAndRefreshToken($email, $password)
    {
        $pClient = PClient::where('password_client', 1)->first();
        $http = new Client;
        $response = $http->post(config('app.url').'/oauth/token', [
            'form_params' => [
                'grant_type' => 'password',
                'client_id' => $pClient->id,
                'client_secret' => $pClient->secret,
                'username' => $email,
                'password' => $password,
                'scope' => '*',
            ],
        ]);

        $result = json_decode((string) $response->getBody(), true);
        return $result;
    }

     /**
     * Refresh Token Function.
     */
    public function getTokenAndRefreshToken($refresh_token)
    {
        $pClient = PClient::where('password_client', 1)->first();
        $http = new Client;
        $response = $http->post(config('app.url').'/oauth/token', [
            'form_params' => [
                'grant_type' => 'refresh_token',
                'refresh_token' => $refresh_token,
                'client_id' => $pClient->id,
                'client_secret' => $pClient->secret,
                'scope' => '*',
            ],
        ]);

        $result = json_decode((string) $response->getBody(), true);
        return $result;
    }
}

{% endhighlight %}

pada method pertama yaitu getFirstTokenAndRefreshToken() kita membutuhkan password dan email sebagai parameter, sedangkan pada method yang kedua  getTokenAndRefreshToken() kita hanya membutuhkan refresh_token.

Kemudian kita buat file LoginController.php dengan isi file sebagai berikut :

{% highlight php %}

<?php

namespace App\Http\Controllers;

use Illuminate\Http\Response;
use Illuminate\Http\Request;
use App\CustomTraits\PassportTokenTrait;
use Auth;

class LoginController extends Controller
{
    use PassportTokenTrait;

    public function __construct()
    {
        $this->middleware('guest')->except('logout');
    }


    public function login(Request $request)
    {
        if (Auth::attempt(['email' => $request->email, 'password' => $request->password]))
        {
            $token = $this->getFirstTokenAndRefreshToken(request('email'), request('password'));
            return response()->json($token, 200);
        }
        else {
            return response()->json(['message'=>'Unauthorised'], 401);
        }
    }

    public function refresh(Request $request)
    {
        $refresh_token = $request->header('refreshtoken');

        try {
            $token = $this->getTokenAndRefreshToken($refresh_token);
            return response()->json($token, 200);
        } catch (\GuzzleHttp\Exception\BadResponseException $e) {
            return response()->json("unauthorized", 401);
        }
    }

}

{% endhighlight %}

selanjutnya kita buat UserController.php, yang untuk mengaksesnya butuh autorisasi, isi dari file UserController adalah sebagai berikut :

{% highlight php %}

<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\Response;

use App\User;

class UserController extends Controller
{

    public function index()
    {
        $user = User::all();
        return response()->json($user, 200);
    }

}


{% endhighlight %}

setelah semua kita buat, kita edit file routes/api.php untuk membuat route yang kita perlukan pada autentikasi aplikasi laravel ini. file api.php kita edit menjadi sebagai berikut :

{% highlight php %}

Route::post('login', 'LoginController@login');
Route::post('refresh', 'LoginController@refresh');

// Protected Route
Route::group(['prefix' => 'v1/user','middleware' => 'auth:api'], function() {
    Route::get('/', 'UserController@index');
});

{% endhighlight %}

Demikian Tutorial dari saya, semoga bermanfaat, apabila ada yang kurang jelas, anda bisa menonton video tentang ini melalui link di bawah ini :

{% include youtubePlayer.html id=page.youtubeId %}