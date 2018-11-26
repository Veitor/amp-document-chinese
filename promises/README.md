---
layout: docs
title: Promises
permalink: /promises/
---
`Promise`是一个代表了异步操作结果的对象。其有三种状态：

 - **Success**: promise处理成功
 - **Failure**: promise处理失败
 - **Pending**: promise还未被处理

一个成功的promise处理结果就像是同步操作中的返回值，而一个失败的promise处理结果就像是抛出了一个异常。

Promise是异步应用程序里并发操作的基本单元，在Amp中其实现了`Amp\Promise`接口。这些对象应该被看作为一些占位符或被看作为一些无法立刻完成的任务。

另一种处理异步API的方式是使用回调，这些回调在操作的一开始就被传入。

```php
doSomething(function ($error, $value) {
    if ($error) {
        /* ... */
    } else {
        /* ... */
    }
});
```

回调方式有几个缺点：

- 传入回调函数以及根据第一个操作的结果做进一步操作会很快变得混乱。
- 需要一个明确的回调函数作为方法的参数，并且回调函数的返回值无法被使用。没有回调函数就不能使用该API。

这正是promise发挥作用的地方。promise是被返回的简单占位符，并且promise允许一个或多个回调函数注册进来。

```php
doSomething()->onResolve(function ($error, $value) {
    if ($error) {
        /* ... */
    } else {
        /* ... */
    }
});
```

乍一眼看这样似乎并没有多大好处，我们只是调整了回调函数。但事实上已经做了很多事。我们现在可以编写像 [`Amp\Promise\all()`](https://amphp.org/amp/promises/combinators#all) 这样的辅助函数，函数订阅了几个promise占位符并将它们合并在一起。我们不需要去编写任何复杂的代码来合并处理几个回调函数的结果。

但是promise最重要的改进是它们允许用协程（[coroutines](https://amphp.org/amp/coroutines/)）来编写，完全不需要任何回到函数。

协程利用了PHP的生成器（generator）。每一次promise被yield返回，协程都会订阅promise，并且一旦promise被处理该协程会自动继续执行。当promise处理成功时，协程会使用[`Generator::send()`](https://secure.php.net/generator.send)发送处理结果到生成器中。当处理失败时，协程将会使用[`Generator::throw()`](https://secure.php.net/generator.throw)抛出异常到生成器中。这几乎像写同步代码一样来写异步代码。

{:.note}
> Amp's `Promise` interface **does not** conform to the "Thenables" abstraction common in JavaScript promise implementations. Chaining `.then()` calls is a suboptimal method for avoiding callback hell in a world with generator coroutines. Instead, Amp utilizes PHP generators as described above.
>
> However, as ReactPHP is another wide-spread implementation, we also accept any `React\Promise\PromiseInterface` where we accept instances of `Amp\Promise`. In case of custom implementations not implementing `React\Promise\PromiseInterface`, `Amp\Promise\adapt()` can be used to adapt any object having a `then` or `done` method.

## Promise Consumption

```php
interface Promise {
    public function onResolve(callable $onResolve);
}
```

In its simplest form the `Amp\Promise` aggregates callbacks for dealing with results once they eventually resolve. While most code will not interact with this API directly thanks to [coroutines](../coroutines/), let's take a quick look at the one simple API method exposed on `Amp\Promise` implementations:

| Parameter    | Callback Signature                         |
| ------------ | ------------------------------------------ |
| `$onResolve` | `function ($error = null, $result = null)` |

`Amp\Promise::onResolve()` accepts an error-first callback. This callback is responsible for reacting to the eventual result represented by the promise placeholder. For example:

```php
<?php

$promise = someFunctionThatReturnsAPromise();
$promise->onResolve(function (Throwable $error = null, $result = null) {
    if ($error) {
        printf(
            "Something went wrong:\n%s\n",
            $error->getMessage()
        );
    } else {
        printf(
            "Hurray! Our result is:\n%s\n",
            print_r($result, true)
        );
    }
});
```

Those familiar with JavaScript code generally reflect that the above interface quickly devolves into ["callback hell"](http://callbackhell.com/), and they're correct. We will shortly see how to avoid this problem in the [coroutines](../coroutines/README.md) section.

## Promise Creation

Promises can be created in several different ways. Most code will use [`Amp\call()`](https://amphp.org/amp/coroutines/helpers#call) which takes a function and runs it as coroutine if it returns a `Generator`.

### Success and Failure

Sometimes values are immediately available. This might be due to them being cached, but can also be the case if an interface mandates a promise to be returned to allow for async I/O but the specific implementation always having the result directly available. In these cases `Amp\Success` and `Amp\Failure` can be used to construct an immediately resolved promise. `Amp\Success` accepts a resolution value. `Amp\Failure` accepts an exception as failure reason.

### Deferred

{:.note}
> The `Deferred` API described below is an advanced API that many applications probably don't need. Use [`Amp\call()`](https://amphp.org/amp/coroutines/helpers#call) or [promise combinators](https://amphp.org/amp/promises/combinators) instead where possible.

`Amp\Deferred` is the abstraction responsible for resolving future values once they become available. A library that resolves values asynchronously creates an `Amp\Deferred` and uses it to return an `Amp\Promise` to API consumers. Once the async library determines that the value is ready it resolves the promise held by the API consumer using methods on the linked promisor.

```php
final class Deferred
{
    public function promise(): Promise;
    public function resolve($result = null);
    public function fail(Throwable $error);
}
```

#### `promise()`

Returns the corresponding `Promise` instance. `Deferred` and `Promise` are separated, so the consumer of the promise can't fulfill it. You should always return `$deferred->promise()` to API consumers. If you're passing `Deferred` objects around, you're probably doing something wrong.

#### `resolve()`

Resolves the promise with the first parameter as value, otherwise `null`. If a `Amp\Promise` is passed, the resolution will wait until the passed promise has been resolved. Invokes all registered `Promise::onResolve()` callbacks.

#### `fail()`

Makes the promise fail. Invokes all registered `Promise::onResolve()` callbacks with the passed `Throwable` as `$error` argument.

Here's a simple example of an async value producer `asyncMultiply()` creating a deferred and returning the associated promise to its API consumer.

```php
<?php // Example async producer using promisor

use Amp\Loop;

function asyncMultiply($x, $y)
{
    // Create a new promisor
    $deferred = new Amp\Deferred;

    // Resolve the async result one second from now
    Loop::delay($msDelay = 1000, function () use ($deferred, $x, $y) {
        $deferred->resolve($x * $y);
    });

    return $deferred->promise();
}

$promise = asyncMultiply(6, 7);
$result = Amp\Promise\wait($promise);

var_dump($result); // int(42)
```