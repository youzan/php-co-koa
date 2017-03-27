## 生成器

统一\Generator接口，由于内部隐式rewind， 需要先调用current() 获取当前value， 而send会直接跳到第二次yield;

1. [send方法说明](http://php.net/manual/en/generator.send.php)
2. [生成器说明](http://www.laruence.com/2015/05/28/3038.html)


```php
<?php

class Gen
{
    public $isfirst = true;
    public $generator;

    public function __construct(\Generator $generator)
    {
        $this->generator = $generator;
    }

    public function valid()
    {
        return $this->generator->valid();
    }

    public function send($value = null)
    {
        if ($this->isfirst) {
            $this->isfirst = false;
            return $this->generator->current();
        } else {
            return $this->generator->send($value);
        }
    }
}
```