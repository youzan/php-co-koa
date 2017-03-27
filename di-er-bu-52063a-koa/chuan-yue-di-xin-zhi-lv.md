## 穿越地心之旅

(如果你了解洋葱圈模型, 略过本小节)

首先让我们对洋葱圈模型有一个直观的认识,

[地球构造](https://zh.wikipedia.org/wiki/%E5%9C%B0%E7%90%83%E6%A7%8B%E9%80%A0)

> 物理学上,地球可划分为岩石圈、软流层、地幔、外核和内核5层。

> 化学上,地球被划分为地壳、上地幔、下地幔、外核和内核。地质学上对地球各层的划分

演示Koa的中间件之前, 我们用函数来描述一场穿越地心之旅:

------------------------------------------------------

```php
<?php
function crust($next)
{
    echo "到达<地壳>\n";
    $next();
    echo "离开<地壳>\n";
}

function upperMantle($next)
{
    echo "到达<上地幔>\n";
    $next();
    echo "离开<上地幔>\n";
}

function mantle($next)
{
    echo "到达<下地幔>\n";
    $next();
    echo "离开<下地幔>\n";
}

function outerCore($next)
{
    echo "到达<外核>\n";
    $next();
    echo "离开<外核>\n";
}

function innerCore($next)
{
    echo "到达<内核>\n";
}

// 我们逆序组合组合, 返回入口
function makeTravel(...$layers)
{
    $next = null;
    $i = count($layers);
    while ($i--) {
        $layer = $layers[$i];
        $next = function() use($layer, $next) {
            // 这里next指向穿越下一次的函数, 作为参数传递给上一层调用
            $layer($next);
        };
    }
    return $next;
}


// 我们途径 crust -> upperMantle -> mantle -> outerCore -> innerCore 到达地心 
// 然后穿越另一半球  -> outerCore -> mantle -> upperMantle -> crust

$travel = makeTravel("crust", "upperMantle", "mantle", "outerCore", "innerCore");
$travel(); // output:
/*
到达<地壳>
到达<上地幔>
到达<下地幔>
到达<外核>
到达<内核>
离开<外核>
离开<下地幔>
离开<上地幔>
离开<地壳>
*/
```

------------------------------------------------------

```php
<?php

function outerCore1($next)
{
    echo "到达<外核>\n";
    // 我们放弃内核, 仅仅绕外壳一周, 从另一侧返回地表
    // $next();
    echo "离开<外核>\n";
}


$travel = makeTravel("crust", "upperMantle", "mantle", "outerCore1", "innerCore");
$travel(); // output:
/*
到达<地壳>
到达<上地幔>
到达<下地幔>
到达<外核>
离开<外核>
离开<下地幔>
离开<上地幔>
离开<地壳>
*/
```

------------------------------------------------------

```php
<?php
function innerCore1($next)
{
    // 我们到达内核之前遭遇了岩浆,计划终止,等待救援
    throw new \Exception("岩浆");
    echo "到达<内核>\n";
}


$travel = makeTravel("crust", "upperMantle", "mantle", "outerCore", "innerCore1");
$travel(); // output:
/*
到达<地壳>
到达<上地幔>
到达<下地幔>
到达<外核>
Fatal error: Uncaught exception 'Exception' with message '岩浆'
*/
```

------------------------------------------------------

```php
<?php

function mantle1($next)
{
    echo "到达<下地幔>\n";
    // 我们在下地幔的救援团队及时赶到 (try catch)
    try {
        $next();
    } catch (\Exception $ex) {
        echo "遇到", $ex->getMessage(), "\n";
    }
    // 我们仍旧没有去往内核, 绕道对端下地幔, 返回地表
    echo "离开<下地幔>\n";
}


$travel = makeTravel("crust", "upperMantle", "mantle1", "outerCore", "innerCore1");
$travel(); // output:
/*
到达<地壳>
到达<上地幔>
到达<下地幔>
到达<外核>
遇到岩浆
离开<下地幔>
离开<上地幔>
离开<地壳>
*/
```

------------------------------------------------------

```php
<?php

function upperMantle1($next)
{
    // 我们放弃对去程上地幔的暂留
    // echo "到达<上地幔>\n";
    $next();
    // 只在返程时暂留
    echo "离开<上地幔>\n";
}

function outerCore2($next)
{
    // 我们决定只在去程考察外核
    echo "到达<外核>\n";
    $next();
    // 因为温度过高,去程匆匆离开外壳
    // echo "离开<外核>\n";
}


$travel = makeTravel("crust", "upperMantle1", "mantle1", "outerCore2", "innerCore1");
$travel(); // output:
/*
到达<地壳>
到达<上地幔>
到达<下地幔>
到达<外核>
遇到岩浆
离开<下地幔>
离开<上地幔>
离开<地壳>
*/
```
