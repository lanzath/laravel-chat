<p align="center"><a href="https://laravel.com" target="_blank"><img src="https://raw.githubusercontent.com/laravel/art/master/logo-lockup/5%20SVG/2%20CMYK/1%20Full%20Color/laravel-logolockup-cmyk-red.svg" width="400"></a></p>

<p align="center">
<a href="https://travis-ci.org/laravel/framework"><img src="https://travis-ci.org/laravel/framework.svg" alt="Build Status"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://poser.pugx.org/laravel/framework/d/total.svg" alt="Total Downloads"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://poser.pugx.org/laravel/framework/v/stable.svg" alt="Latest Stable Version"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://poser.pugx.org/laravel/framework/license.svg" alt="License"></a>
</p>

# Criando notificações  e eventos real-time com Laravel.

No exemplo criado foi utilizado como substituto ao pusher o pacote laravel websockets, mas também é possível implementar com o pusher conforme será documentado aqui.



Instalação:

- Laravel Websockets
- Pusher
- laravel-echo

Primeiramente para o *backend* é instalado o [laravel-websockets](https://beyondco.de/docs/laravel-websockets/getting-started/introduction) e [pusher](https://pusher.com/) caso seja utilizado o laravel-websockets, se for feito com o pusher apenas o pusher deve ser instalado.

`composer require beyondcode/laravel-websockets "1.9"`

`composer require pusher/pusher-php-server "^4.1"`

Nesse exemplo e teste, utilizei a versão **4.1** do *pusher* e a **1.9** do *laravel-websockets.*

Configurações
`.env`

Deve ser indicado no **.env** da aplicação `BROADCAST_DRIVER=pusher`, lembrando que o laravel-websockets é uma alternativa ao pusher.

Também é necessário configurar as seguintes chaves:
``` 
PUSHER_APP_ID=12345
PUSHER_APP_KEY=12345
PUSHER_APP_SECRET=12345
PUSHER_APP_CLUSTER=mt1
```
Para o *Laravel-websockets* são utilizados **valores quaisquer**, caso a aplicação use o *pusher* deve ser **gerada as chaves no próprio site**.


Também ao utilizar o *laravel-websockets* ele vem com alguns arquivos de *migration* que devem ser publicados no projeto com o comando 

`php artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="migrations"` 

Migrá-los com `php artisan migrate`

E publicar as configurações:

`php artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="config"
`

Lembrando que só é necessário essas publicações caso seja utilizado o **laravel-websockets**

Após os websockets estarem devidamente configurados basta rodar o comando 

`php artisan websockets:serve`


As notificações podem ser criadas como **eventos** ou como **notificações**, foi abordado ambos os métodos nessa documentação.

### Eventos

Criando um evento:

`php artisan make:event RealTimeMessage` 

```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Queue\SerializesModels;

class RealTimeMessage implements ShouldBroadcast
{
    use SerializesModels;

    public string $message;

    public function __construct(string $message)
    {
        $this->message = $message;
    }

	// Canal no qual o evento será transmitido.
    public function broadcastOn(): Channel
    {
        return new Channel('events');
    }
}
```

Antes de transmitir o evento deve ser configurado o arquivo `broadcasting.php`, na chave `options` no array de configurações deve estar as seguintes propriedades (Válido para utilização do **laravel-websockets**):
```php
'options' => [
    'cluster' => env('PUSHER_APP_CLUSTER'),
    'encrypted' => false,
    'host' => '127.0.0.1',
    'port' => 6001,
    'scheme' => 'http'
],
```


#### Frontend
Como dependências front-end para os eventos e notificações em tempo real, foram instaladas o **laravel-echo** e **pusher-js**
`npm install --save-dev laravel-echo pusher-js`

Configuração (Laravel-websocket):
```js
import Echo from 'laravel-echo';

window.Pusher = require('pusher-js');

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: process.env.MIX_PUSHER_APP_KEY,
    cluster: process.env.MIX_PUSHER_APP_CLUSTER,
    forceTLS: false,
    wsHost: window.location.hostname,
    wsPort: 6001,
});
```

Script para "ouvir" novos eventos:
```js
Echo.channel('events')
        .listen('RealTimeMessage', (e) => console.log('RealTimeMessage: ' + e.message));
```

Após feitas essas configurações pode-se emitir um evento do backend pelo próprio PHP artisan tinker ou chamá-lo em alguma controller utilizando o heper `event()`.

Exemplo:
`event(new App\Events\RealTimeMessage('Hello World'));`



#### Eventos privados
No exemplo acima foi feito um meio de transmissão de evento de modo público.
Para ser feita a transmissão de maneira privativa é necessário que haja usuário logado e configurações adicionais.

Primeiramente, na classe do evento, deve ser trocado o Channel por PrivateChannel:
```php
public function broadcastOn(): Channel
{
    return new PrivateChannel('events');
}
```

Assim como o código js também deve ser atualizado para indicar que está sendo usado um canal privado.
```js
Echo.private('events')
    .listen('RealTimeMessage', (e) => console.log('Private RealTimeMessage: ' + e.message));
```

Também deve ser ativado o provider de Broadcast em `app.php` nas configurações, basta apenas descomentar o código: 
`App\Providers\BroadcastServiceProvider::class,`

Permitindo o acesso de usuario a um canal, em `routes/channels.php`:
```php
Broadcast::channel('events', function ($user) {
    return true;
});
```

Após essas configurações, é possível receber os eventos em tempo real novamente.


### Notificações
Criando uma nova notificação:
`php artisan make:notification RealTimeNotification`


```php
<?php

namespace App\Notifications;

use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Notifications\Messages\BroadcastMessage;
use Illuminate\Notifications\Notification;

class RealTimeNotification extends Notification implements ShouldBroadcast
{

    public string $message;

    public function __construct(string $message)
    {
        $this->message = $message;
    }

    public function via($notifiable): array
    {
        return ['broadcast'];
    }

	// Como notificações sempre estão conectadas a models que contenham notifiable
    // temos acesso ao método toBroadcast, assim podemos passar o id na mensagem enviada.
    public function toBroadcast($notifiable): BroadcastMessage
    {
        return new BroadcastMessage([
            'message' => "$this->message (User $notifiable->id)"
        ]);
    }
}
```

Exemplo de `trigger`de notificação:
`$user->notify(new App\Notifications\RealTimeNotification('Hello World'));`


Padrão de nome de canal para notificaçõe `App.Models.User.{id}`
Sendo assim, nosso channel em `routes/channels.php`
```php
Broadcast::channel('App.Models.User.{id}', function ($user, $id) {
    return (int) $user->id === (int) $id;
});
```

e no arquivo de javascript:

```js
Echo.private('App.Models.User.1')
    .notification((notification) => {
        console.log(notification.message);
    });
```


## About Laravel

Laravel is a web application framework with expressive, elegant syntax. We believe development must be an enjoyable and creative experience to be truly fulfilling. Laravel takes the pain out of development by easing common tasks used in many web projects, such as:

- [Simple, fast routing engine](https://laravel.com/docs/routing).
- [Powerful dependency injection container](https://laravel.com/docs/container).
- Multiple back-ends for [session](https://laravel.com/docs/session) and [cache](https://laravel.com/docs/cache) storage.
- Expressive, intuitive [database ORM](https://laravel.com/docs/eloquent).
- Database agnostic [schema migrations](https://laravel.com/docs/migrations).
- [Robust background job processing](https://laravel.com/docs/queues).
- [Real-time event broadcasting](https://laravel.com/docs/broadcasting).

Laravel is accessible, powerful, and provides tools required for large, robust applications.

## Learning Laravel

Laravel has the most extensive and thorough [documentation](https://laravel.com/docs) and video tutorial library of all modern web application frameworks, making it a breeze to get started with the framework.

If you don't feel like reading, [Laracasts](https://laracasts.com) can help. Laracasts contains over 1500 video tutorials on a range of topics including Laravel, modern PHP, unit testing, and JavaScript. Boost your skills by digging into our comprehensive video library.

## Laravel Sponsors

We would like to extend our thanks to the following sponsors for funding Laravel development. If you are interested in becoming a sponsor, please visit the Laravel [Patreon page](https://patreon.com/taylorotwell).

### Premium Partners

- **[Vehikl](https://vehikl.com/)**
- **[Tighten Co.](https://tighten.co)**
- **[Kirschbaum Development Group](https://kirschbaumdevelopment.com)**
- **[64 Robots](https://64robots.com)**
- **[Cubet Techno Labs](https://cubettech.com)**
- **[Cyber-Duck](https://cyber-duck.co.uk)**
- **[Many](https://www.many.co.uk)**
- **[Webdock, Fast VPS Hosting](https://www.webdock.io/en)**
- **[DevSquad](https://devsquad.com)**
- **[OP.GG](https://op.gg)**

## Contributing

Thank you for considering contributing to the Laravel framework! The contribution guide can be found in the [Laravel documentation](https://laravel.com/docs/contributions).

## Code of Conduct

In order to ensure that the Laravel community is welcoming to all, please review and abide by the [Code of Conduct](https://laravel.com/docs/contributions#code-of-conduct).

## Security Vulnerabilities

If you discover a security vulnerability within Laravel, please send an e-mail to Taylor Otwell via [taylor@laravel.com](mailto:taylor@laravel.com). All security vulnerabilities will be promptly addressed.

## License

The Laravel framework is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).
