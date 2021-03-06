laravel-grip
================

Author: Konstantin Bokarius <kon@fanout.io>

A Laravel GRIP library.

License
-------

laravel-grip is offered under the MIT license. See the LICENSE file.

Requirements
------------

* openssl
* curl
* pthreads (required for asynchronous publishing)
* laravel >=5.0
* fanout/gripcontrol >=2.0.0 (retrieved automatically via Composer)

Installation
------------

Using Composer:

```sh
composer require fanout/laravel-grip
```

Manual: ensure that php-gripcontrol has been included and require the following files in laravel-grip:

```PHP
require 'laravel-grip/src/gripmiddleware.php';
require 'laravel-grip/src/websocketcontext.php';
require 'laravel-grip/src/laravelgrip.php';
require 'laravel-grip/src/nonwebsocketrequestexception.php';
```

Asynchronous Publishing
-----------------------

In order to make asynchronous publish calls pthreads must be installed. If pthreads is not installed then only synchronous publish calls can be made. To install pthreads recompile PHP with the following flag: '--enable-maintainer-zts'

Also note that since a callback passed to the publish_async methods is going to be executed in a separate thread, that callback and the class it belongs to are subject to the rules and limitations imposed by the pthreads extension.

See more information about pthreads here: http://php.net/manual/en/book.pthreads.php

Usage
-----

Set grip_proxies in your application configuration:

```
# pushpin and/or fanout.io is used for sending realtime data to clients
'grip_proxies' => [
    # pushpin
    [
        'control_uri' => 'http://localhost:5561',
        'key' => 'changeme'
    ]
    # fanout.io
    #[
    #    'control_uri' => 'https://api.fanout.io/realm/your-realm',
    #    'control_iss' => 'your-realm',
    #    'key' => base64_decode('your-realm-key')
    #]
]
```

If it's possible for clients to access the Laravel app directly, without necessarily going through the GRIP proxy, then you may want to avoid sending GRIP instructions to those clients. An easy way to achieve this is with the grip_proxy_required setting. If set, then any direct requests that trigger a GRIP instruction response will be given a 501 Not Implemented error instead.

```
'grip_proxy_required' => true
```

To prepend a fixed string to all channels used for publishing and subscribing, set grip_prefix in your configuration:

```
'grip_prefix' => '<prefix>'
```

You can also set any other EPCP servers that aren't necessarily proxies with publish_servers:

```
'publish_servers' => [
    [
        'uri' => 'https://api.fanout.io/realm/your-realm',
        'iss' => 'your-iss',
        'key' => 'your-key'
    ]
]
```

This library also comes with a middleware class that you should use. The middleware will parse the Grip-Sig header in any requests in order to detect if they came from a GRIP proxy, and it will apply any hold instructions when responding. Additionally, the middleware handles WebSocket-Over-HTTP processing so that WebSockets managed by the GRIP proxy can be controlled via HTTP responses from the Laravel application.

The middleware should be placed as early as possible in the proessing order, so that it can collect all response headers and provide them in a hold instruction if necessary.

To register the middle add it to the global middleware stack in app/Http/Kernel.php:

```php
protected $middleware = [
    ...
    \LaravelGrip\GripMiddleware::class,
    ...
];
```

Note that the built-in VerifyCsrfToken middleware in Laravel will throw a TokenMismatchException for POST requests coming from GRIP proxies. See this post regarding solutions for avoiding this exception: <http://stackoverflow.com/questions/29192806/laravel-curl-post-throwing-tokenmismatchexception>

Example controller:

```php
Route::get('/', [function () {;
    # if the request didn't come through a GRIP proxy, throw 501
    if (!LaravelGrip\is_grip_proxied())
        return new Illuminate\Http\Response(
                'Not implemented.\n', 501);

    # subscribe every incoming request to a channel in stream mode
    LaravelGrip\set_hold_stream('<channel>');
    return '[stream open]';
}]);
```

Stateless WebSocket echo service with broadcast endpoint:

```php
Route::post('/', [function () {
    # reject non-websocket requests
    LaravelGrip\verify_is_websocket();

    # if this is a new connection, accept it and subscribe it to a channel
    $ws = LaravelGrip\get_wscontext();
    if ($ws->is_opening())
    {
        $ws->accept();
        $ws->subscribe('<channel>');
    }

    while ($ws->can_recv())
    {
        $message = $ws->recv();

        # if return value is nil, then the connection is closed
        if (is_null($message))
        {
            $ws->close();
            break;
        }

        # echo the message
        $ws->send($message);
    }

    return null;
}]);

Route::post('/broadcast', [function () {
    $data = \Request::instance()->getContent();

    # publish data to all clients that are connected to the echo endpoint
    LaravelGrip\publish('<channel>',
        new \GripControl\WebSocketMessageFormat($data));

    return "Ok\n";
}]);
```
