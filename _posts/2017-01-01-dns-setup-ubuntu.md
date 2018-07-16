# 支付网关项目

## 测试

单元测试多的情况下，可能php内存限制不够，也导致速度慢，

可通过以下命令运行，不限制内存，添加` php -d memory_limit=-1 `前缀即可。

php -d memory_limit=-1 ./vendor/bin/phpuni

1. 在项目目录下运行 ./vendor/bin/phpunit

1. 显示代码覆盖率 ./vendor/bin/phpunit --coverage-text 

1. 输出详细代码覆盖情况 php -d memory_limit=-1 ./vendor/bin/phpunit --coverage-html ./coverage  

1. 运行指定类     ./vendor/bin/phpunit --filter ApplyWithdrawRequestTest  

## QA

1. Message was: "Call to undefined function msgpack_pack()”：依赖msgpack拓展。
1. No code coverage driver is available：依赖xdebug拓展。
1. ErrorException: Use of undefined constant MCRYPT_RIJNDAEL_128: 需要mcrypt拓展。