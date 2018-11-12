# PHP相关

php是最好的语言！

## PHP弱类型

* `var_dump('0xABCdef' == ' 0xABCdef');`
    * true (Output for hhvm-3.18.5 - 3.22.0, 7.0.0 - 7.2.0rc4: false)
* `var_dump('0010e2' == '1e3’);`
    * true
* `strcmp([],[])`
    * 0
* `sha1([])`
    * NULL
* `'123' == 123`
* `'abc' == 0`
* `'0x01' == 1`
    * PHP 7.0后，16位字符串不在当成数字（`'0x01 != 1'`）
* `0 == '' == false == NULL`
* `$a = 'a'`
    * `++$a` =>`'b'`
    * `$a+1` => 1

## PHP特性

