# PHP相关

php是最好的语言！

## PHP弱类型
![](../statics/php_equiv.png)

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

## PHP函数特性
### intval
* 四舍五入(不存在的截断)
    * `var_dump(intval('5278.78'))` => 5278
* `intval(012)` => 10
* `intval("012")` => 12

### extract
`#!php int extract ( array &$array [, int $flags = EXTR_OVERWRITE [, string $prefix = NULL ]] )`
