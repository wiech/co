# Co [![Build Status](https://travis-ci.org/mpyw/co.svg?branch=master)](https://travis-ci.org/mpyw/co) [![Coverage Status](https://coveralls.io/repos/github/mpyw/co/badge.svg?branch=master)](https://coveralls.io/github/mpyw/co?branch=master) [![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/mpyw/co/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/mpyw/co/?branch=master)

Asynchronous cURL executor simply based on resource and Generator

| PHP | :question: | Feature Restriction |
|:---:|:---:|:---:|
| 7.0~ | :smile: | Full Support |
| 5.5~5.6 | :anguished: | Generator is not so cool |
| ~5.4 | :boom: | Incompatible |

```php
function curl_init_with(string $url, array $options = [])
{
    $ch = curl_init();
    $options = array_replace([
        CURLOPT_URL => $url,
        CURLOPT_RETURNTRANSFER => true,
    ], $options);
    curl_setopt_array($ch, $options);
    return $ch;
}
function get_xpath_async(string $url) : \Generator
{
    $dom = new \DOMDocument;
    @$dom->loadHTML(yield curl_init_with($url));
    return new \DOMXPath($dom);
}

var_dump(Co::wait([

    'Delay 5 secs' => function () {
        echo "[Delay] I start to have a pseudo-sleep in this coroutine for about 5 secs\n";
        for ($i = 0; $i < 5; ++$i) {
            yield Co::DELAY => 1;
            if ($i < 4) {
                printf("[Delay] %s\n", str_repeat('.', $i + 1));
            }
        }
        echo "[Delay] Done!\n";
    },

    "google.com HTML" => curl_init_with("https://google.com"),

    "Content-Length of github.com" => function () {
        echo "[GitHub] I start to request for github.com to calculate Content-Length\n";
        $content = yield curl_init_with("https://github.com");
        echo "[GitHub] Done! Now I calculate length of contents\n";
        return strlen($content);
    },

    "Save mpyw's Gravatar Image URL to local" => function () {
        echo "[Gravatar] I start to request for github.com to get Gravatar URL\n";
        $src = (yield get_xpath_async('https://github.com/mpyw'))
                 ->evaluate('string(//img[contains(@class,"avatar")]/@src)');
        echo "[Gravatar] Done! Now I download its data\n";
        yield curl_init_with($src, [CURLOPT_FILE => fopen('/tmp/mpyw.png', 'wb')]);
        echo "[Gravatar] Done! Saved as /tmp/mpyw.png\n";
    }

]));
```

The requests are executed as parallelly as possible :smile:  
Note that there is only **1 process** and **1 thread**.

```Text
[Delay] I start to have a pseudo-sleep in this coroutine for about 5 secs
[GitHub] I start to request for github.com to calculate Content-Length
[Gravatar] I start to request for github.com to get Gravatar URL
[Delay] .
[Delay] ..
[GitHub] Done! Now I calculate length of contents
[Gravatar] Done! Now I download its data
[Delay] ...
[Gravatar] Done! Saved as /tmp/mpyw.png
[Delay] ....
[Delay] Done!
array(4) {
  ["Delay 5 secs"]=>
  NULL
  ["google.com HTML"]=>
  string(262) "<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>302 Moved</TITLE></HEAD><BODY>
<H1>302 Moved</H1>
The document has moved
<A HREF="https://www.google.co.jp/?gfe_rd=cr&amp;ei=XXXXXX">here</A>.
</BODY></HTML>
"
  ["Content-Length of github.com"]=>
  int(25534)
  ["Save mpyw's Gravatar Image URL to local"]=>
  NULL
}
```

## Installing

Install via Composer.

```sh
composer require mpyw/co:^1.5
```

And require Composer autoloader in your scripts.

```php
require 'vendor/autoload.php';

use mpyw\Co\Co;
use mpyw\Co\CURLException;
```

## API

### Co::wait()

Wait for all the cURL requests to complete.  
The options will override static defaults.

```php
static Co::wait(mixed $value, array $options = []) : mixed
```

#### Arguments

- **`(mixed)`** __*$value*__<br /> Any values to be parallelly resolved.
- **`(array<string, mixed>)`** __*$options*__<br /> Associative array of options.

| Key | Default | Description |
|:---:|:---:|:---|
| `throw` | **`true`** | Whether to throw or capture `CURLException` or `RuntimeException` on top-level.|
| `pipeline` | **`false`** | Whether to use HTTP/1.1 pipelining.<br />**At most 5** requests for the same destination are bundled into single TCP connection.|
| `multiplex` | **`true`** | Whether to use HTTP/2 multiplexing.<br />**All** requests for the same destination are bundled into single TCP connection.|
| `interval` | **`0.002`** | `curl_multi_select()` timeout seconds. `0` means real-time observation.|
| `concurrency` | **`6`** | Limit of concurrent TCP connections. `0` means unlimited.<br />The value should be within `10` at most. |

- `Throwable` those are not extended from `RuntimeException`, such as `Error` `Exception` `LogicException` are not captured. If you need to capture them, you have to write your own try-catch blocks in your functions.
- HTTP/1.1 pipelining can be used only if the TCP connection is already established and verified that uses keep-alive session. It means that **the first bundle of HTTP/1.1 requests CANNOT be pipelined**. You can use it from second `yield` in `Co::wait()` call.
- To use HTTP/2 multiplexing, you have to build PHP with libcurl 7.43.0+ and `--with-nghttp2`.
- `concurrency` controlling with `pipeline` / `multiplex` CANNOT be correctly driven in **PHP 7.0.6 or before**. You should set higher `concurrency` if you use pipelining or multiplexing in those versions.

#### Return Value

**`(mixed)`**<br />Resolved values; in exception-safe context, it may contain...

- `CURLException` which has been raised internally.
- `RuntimeException` which has been raised by user.

#### Exception

- Throws `CURLException` or `RuntimeException` in exception-unsafe context.

### Co::async()

Execute cURL requests along with `Co::wait()` call, **without waiting** resolved values.  
The options are inherited from `Co::wait()`.  

This method is mainly expected to be used ...

- When you are not interested in responses.
- In `CURLOPT_WRITEFUNCTION` or `CURLOPT_HEADERFUNCTION` callbacks.

```php
static Co::async(mixed $value, mixed $throw = null) : null
```

#### Arguments

- **`(mixed)`** __*$value*__<br /> Any values to be parallelly resolved.
- **`(mixed)`** __*$throw*__<br /> Overrides `throw` in `Co::wait()` options when you passed `true` or `false`.

#### Return Value

**`(null)`**

#### Exception

- `CURLException` or `RuntimeException` can be thrown in exception-unsafe context.<br />Note that you **CANNOT** capture top-level exceptions unless you catch **outside of `Co::wait()` call**.

### Co::setDefaultOptions()<br />Co::getDefaultOptions()

Overrides/gets static default settings.

```php
static Co::setDefaultOptions(array $options) : null
static Co::getDefaultOptions() : array
```

## Rules

### Conversion on Resolving

The all yielded/returned values are resolved by the following rules.  
Yielded values are also resent to the Generator.  
The rules will be applied recursively.

| Before | After |
|:---:|:----:|
|cURL resource|`curl_multi_getconent()` result or `CURLException`|
|Array|Array (with resolved children) or `RuntimeException`|
|Generator Closure<br>Generator| Return value (after all yields done) or `RuntimeException`|

"Generator Closure" means Closure that contains `yield` keywords.

### Exception-safe or Exception-unsafe Priority

#### Context in Generator

**Exception-unsafe** context by default.  
The following `yield` statement specifies exception-safe context.

```php
$results = yield Co::SAFE => [$ch1, $ch2];
```

This is equivalent to:

```php
$results = yield [
    function () use ($ch1) {
        try {
            return yield $ch1;
        } catch (\RuntimeException $e) {
            return $e;
        }
    },
    function () use ($ch2) {
        try {
            return yield $ch2;
        } catch (\RuntimeException $e) {
            return $e;
        }
    },
];
```

#### Context on `Co::wait()`

**Exception-unsafe** context by default.  
The following setting specifies exception-safe context.

```php
$result = Co::wait([$ch1, $ch2], ['throw' => false]);
```

This is equivalent to:

```php
$results = Co::wait([
    function () use ($ch1) {
        try {
            return yield $ch1;
        } catch (\RuntimeException $e) {
            return $e;
        }
    },
    function () use ($ch2) {
        try {
            return yield $ch2;
        } catch (\RuntimeException $e) {
            return $e;
        }
    },
]);
```

#### Context on `Co::async()`

Contexts are **inherited** from `Co::wait()`.  
The following setting overrides parent context as exception-safe.

```php
Co::async($value, false);
```

The following setting overrides parent context as exception-unsafe.

```php
Co::async($value, true);
```

### Pseudo-sleep for Each Coroutine

The following `yield` statements delay the coroutine processing:

```php
yield Co::DELAY => $seconds
yield Co::SLEEP => $seconds  # Alias
```

### Comparison with Generators of PHP7.0+ or PHP5.5~5.6

#### `return` Statements

PHP 7.0+:

```php
yield $foo;
yield $bar;
return $baz;
```

PHP 5.5~5.6:

```php
yield $foo;
yield $bar;
yield Co::RETURN_WITH => $baz;
```

Although experimental aliases `Co::RETURN_` `Co::RET` `Co::RTN` are provided,  
**`Co::RETURN_WITH`** is recommended in terms of readability.

#### `yield` Statements with Assignment

PHP 7.0+:

```php
$a = yield $foo;
echo yield $bar;
```

PHP 5.5~5.6:

```php
$a = (yield $foo);
echo (yield $bar);
```

#### `finally` Statements

Be careful that `return` triggers `finally` while `yield Co::RETURN_WITH =>` does not.

```php
try {
    return '...';
} finally {
    // Reachable
}
```

```php
try {
    yield Co::RETURN_WITH => '...';
} finally {
    // Unreachable
}
```
