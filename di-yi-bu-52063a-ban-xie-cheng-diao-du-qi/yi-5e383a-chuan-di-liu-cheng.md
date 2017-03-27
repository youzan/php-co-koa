## 异常: 传递流程

基于上述注释观察异常传递流程;

```php
<?php

function g1()
{
    throw new \Exception();
    yield;
}
// a2 -> b2 ->
(new AsyncTask(g1()))->begin();


function g2()
{
    yield;
    throw new \Exception();
}
// a2 (-> a4 -> a2) -> b2 -> b2 ->
(new AsyncTask(g2()))->begin();


function g3()
{
    yield;
    throw new \Exception();
}
// a2 (-> a4 -> a2) -> b2 -> b2 ->
(new AsyncTask(g3()))->begin();


function g4()
{
    yield;
    yield;
    throw new \Exception();
}
// a2 (-> a4 -> a2) (-> a4 -> a2) -> b2 -> b2 -> b2 ->
(new AsyncTask(g4()))->begin();


function g5()
{
    throw new \Exception();
    /** @noinspection PhpUnreachableStatementInspection */
    yield;
}
function g7()
{
    yield g5();
}
// (a2 -> a3) -> a2 (-> b2 -> b1 -> c) -> b2 ->
(new AsyncTask(g7()))->begin();


function g6()
{
    yield;
    throw new \Exception();
}
function g8()
{
    yield g6();
}
// (a2 -> a3) -> a2 -> a4 -> a2 -> b2 -> b2 -> b1 -> c -> b2 ->
(new AsyncTask(g8()))->begin();


function g9()
{
    try {
        yield g5();
    } catch (\Exception $ex) {

    }
}
// a2 -> a3 -> a2 -> b2 -> b1 -> c ->
(new AsyncTask(g9()))->begin();
```
