# PHP RFC: User Defined Operator Overloads
  * Version: 0.5
  * Date: 2021-08-14
  * Author: Jordan LeDoux, jordan.ledoux@gmail.com
  * Status: Under Discussion
  * First Published at: http://wiki.php.net/rfc/user_defined_operator_overloads

## Introduction

PHP allows classes written in C to implement operator overloading. This is used in the DateTime library for easy comparison of DateTime objects:

```php
function checkDateIsOverOneHourInFuture(DateTimeInterface $date)
{
    $one_hour_in_future = new DateTime("+1 hour");

    if ($date <= $one_hour_in_future) {
        return false;
    }
    return true;
}
```

Operator overloading is used by GMP to allow easier to read code. These two pieces of code are equivalent:

```
// without operator overloading
$result = gmp_mod(
    gmp_add(
        gmp_mul($c0, gmp_mul($ms0, gmp_invert($ms0, $n0))),
        gmp_add(
            gmp_mul($c1, gmp_mul($ms1, gmp_invert($ms1, $n1))),
            gmp_mul($c2, gmp_mul($ms2, gmp_invert($ms2, $n2)))
        )
    ),
    gmp_mul($n0, gmp_mul($n1, $n2))
);
```

```
// with operator overloading
$result = (
    $c0 * $ms0 * gmp_invert($ms0, $n0)
  + $c1 * $ms1 * gmp_invert($ms1, $n1)
  + $c2 * $ms2 * gmp_invert($ms2, $n2)
) % ($n0 * $n1 * $n2);
```

Even without understanding what the above code does (it's an excerpt from a Coppersmith attack on RSA), it should be obvious that the second code is a lot clearer. It makes the structure of the code immediately clear (three multiplications are summed up and the modulus is taken), whereas the function-based code actively hides any structure in the code. For mathematical operations infix notation just comes a lot more naturally.

## Proposal

This RFC proposes allowing classes written in PHP (aka userland) to implement operator overloading, for a limited set of operators, similar to how it is allowed for classes written in C.

### Add support for operators

For most operators, the magic methods have the form `function __op($other, bool $left)` and so an implementation to support the add operator would look like:

```
class Money
{
    public function __construct(
        readonly int  $unit,
        readonly string $currency) {}

    public function __add(Money $other, bool $left): Money
    {
        if ($this->currency !== $other->currency) {
            throw new Exception("Only money in the same currency can be added.");
        }
        return new Money($this->unit + $other->unit, $this->currency);
    }
}
```

### Operator order and retrying

If the left operand doesn't support the operation (i.e. it doesn't implement the relevant magic method) then the engine will retry the operation using the right operand.

```php
<?php

class Number {
    function __construct(int $value) {}

    function __add(Number|int $other, bool $left): Number
    {
        if (is_int($other)) {
            return new Number($this->value + $other);
        } else {
            return new Number($this->value + $other->value);
        }
    }
}

$num = new Number(5);

$val1 = $num + 1;
// this is equivalent to
// $val1 = $num->__add(1, true);

$val2 = 1 + $num;
// this is equivalent to
// $num->__add(1, false);
```

If the called object is the left operand, then $left is true. If the called object is the right operand, then $left is false.

TODO - needs an example where someone would check $left

### Operators supported

This RFC proposes only a subset of the operators in PHP are supported for operator overloading. See also the sections below on the excluded logic and equality operators, and the implied operators.

The list of supported operations and their signatures are:

| Operator | Magic Method |
|---|---|
| + | __add($other, bool $left): mixed |
| - | __sub($other, bool $left): mixed |
| * | __mul($other, bool $left): mixed |
| /  | __div($other, bool $left): mixed |
| %  | __mod($other, bool $left): mixed |
| &  | __bitwiseAnd($other, bool $left): mixed |
| \| | __bitwiseOr($other, bool $left): mixed |
| ^  | __bitwiseXor($other, bool $left): mixed |
| ~  | __bitwiseNot(): mixed; |
| << | __bitwiseShiftLeft($other, bool $left): mixed |
| >> | __bitwiseShiftRight($other, bool $left): mixed |
| ** | __pow($other, bool $left): mixed |
| == | public function __equals(mixed $other): mixed |
| <=> | public function __compareTo(mixed $other): int |

The magic methods can be implemented with parameter and return types to narrow the type accepted e.g.

```
class Money
{
    public function __construct(
        readonly int  $unit,
        readonly string $currency) {}

    public function __add(Money $other, bool $left): Money
    {
        if ($this->currency !== $other->currency) {
            throw new MoneyException("Cannot add Money in different currencies.");
        }
        return new Money($this->unit + $other->unit, $this->currency);
    }
}
```

### Implied operators

Many expressions in PHP can be result to simpler forms.

| Operator | Implied As | Method |
|----------|------------|--------|
| $a += $b	| $a = $a + $b | __add |
| $a -= $b	| $a = $a - $b | __sub |
| $a *= $b	| $a = $a * $b | __mul |
| $a /= $b	| $a = $a / $b | __div |
| $a %= $b	| $a = $a % $b | __mod |
| $a **= $b	| $a = $a ** $b | __pow |
| $a &= $b	| $a = $a & $b | __bitwiseAnd |
| $a |= $b	| $a = $a | $b | __bitwiseOr |
| $a ^= $b	| $a = $a ^ $b | __bitwiseXor |
| $a <<= $b	| $a = $a << $b | __bitwiseShiftLeft |
| $a >>= $b	| $a = $a >> $b | __bitwiseShiftRight |
| $a != $b	| !($a == $b) | __equals |
| $a < $b	| ($a <=> $b) == -1 | __compareTo |
| $a <= $b	| ($a <=> $b) < 1 | __compareTo |
| $a > $b	| ($a <=> $b) == 1 | __compareTo |
| $a >= $b	| ($a <=> $b) > -1 | __compareTo |
| ++$a      | $a = $a + 1 | __add |
| $a++      | $a = $a + 1 | __add |
| --$a      | $a = $a - 1 | __sub |
| $a--      | $a = $a - 1 | __sub |
| -$a       | $a = -1 * $a |  __mul |

All those expressions that reduced to implied forms will work, but don't have individual operators.

### Notable operators

Most of the operators follow the form `function __op($other, bool $left): mixed` and those all behave in the same way. There are a few operators that will have a different signature, and/or behave differently.

#### __bitwiseNot

The bitwise not operator is unary (acts on a single operand) and so has the signature `__bitwiseNot(): mixed` i.e. that magic method has no parameters.

#### __equals

Comparisons have a reflection relationship instead of a commutative one, the $left argument is omitted as it could only be used for evil (making `$x == $y` have a different result than `$y == $x`.

Comparison operators do not throw the InvalidOperator error when unimplemented. Instead, the PHP engine falls back to existing comparison logic in the absence of an override for a given class.

The magic methods for the comparison operators have the additional restriction of returning bool or int instead of mixed.

#### Comparison operators aka __compareTo

The __compareTo operator requires an int return rather than mixed:

```
 function __compareTo(mixed $other): int
```

Any return value larger than 0 will be normalized to 1, and any return value smaller than 0 will be normalized to -1.

The $left argument is omitted as it could only be used for evil e.g. implementing different comparison logic depending on which side its on. Instead of passing $left the engine will multiply the result of the __compareTo call by (-1) where appropriate:

```
class Number
{
    function __construct(readonly int $value) {}

    public function __compareTo(Number|int $other): int
    {
        if ($other instanceof Number) {
            return $this->value - $other->value; // TODO - check right way round
        }

        return $this->value - $other; // TODO - check right way round
    }
}

$obj = new Number(5);

$less_than = ($obj < 5);
// is equivalent to
$less_than = $obj->__compareTo(5);

$greater_than = 5 > $obj;
// is equivalent to
$greater_than = ($obj->__compareTo(5) * - 1);

```

 By doing * -1, no matter what the implementation is from the PHP, the >, >=, ==, <=, < comparisons are all guaranteed to be consistent regardless of whether the operand is on the left or right side. This avoids situations where both $obj1 > 5 and 5 > $obj1 can return true.

Comparison operators do not throw the InvalidOperator error when unimplemented. Instead, the PHP engine falls back to existing comparison logic in the absence of an override for a given class.

This means that if __equals() is unimplemented but __compareTo() is implemented, the expression $obj == 6 would be evaluated as $obj <=> 6 === 0. As such, for objects which are both equatable and comparable, such as arbitrary precision numbers, it is only necessary to implement __compareTo(). However, for objects that are equatable but not comparable such as complex numbers, both would need an implementation with the __compareTo() explicitly throwing an exception or error. TODO DJA - this sounds suboptimal/weird. aka I don't fully grok it.

### Add InvalidOperatorError

A new InvalidOperatorError error, which extends TypeError/Error will be defined, which will be thrown when an object which does not implement the appropriate operator overload is attempted to be operated on. TODO - this is terrible phrasing.

```
$obj = new StdClass();
$value = $obj + 3;

Uncaught InvalidOperatorError: Unsupported operand types: stdClass + int
```
Currently, this throws a TypeError.

## F.A.Q.

### Won't operator overloading be misused?

Yes.

It is a common pattern when developers (particularly juniors who have not learnt an appropriate level of fear yet) learn about a new feature, they will use it in ways that more senior/experienced developers would consider as 'bad'.

Most of the argument [against operator overloading](https://james-iry.blogspot.com/2009/03/operator-overloading-ad-absurdum.html) boils down to:

> The problem is abuse. Somebody will name something '+' when it has
> nothing to do with the common notion of '+'. The resulting confusion
> is a bigger downside than the benefits of allowing programmers to be
> flexible in naming.

But this is true of any feature in PHP. People are free to write 'getter' methods that mutate an object state rather than just returning a value.

For all valid names X that evoke a common conception, somebody will name something 'X' when it has nothing to do with the common notion of 'X'.

Language design shouldn't focus on preventing people from doing things you disagree with, at the expense of blocking appropriate usage of a feature.

### When to use $left

Not all operations are commutative, for example this stunning matrix example:

```
TODO - add Matrix class example

$result = [1, 0, 1] * $matrix
```

### Why magic methods and not interfaces?

Interfaces wouldn't help write correct code.

For example, a Vector2d and a Money class could both implement an __addable interface, but using __addable as a type check wouldn't prevent users from attempting to add incompatible objects.

```
interface __addable
{
    function __add($other, bool $left): mixed
}

class Money implements __addable {}
class Vector implements __addable {}

function processValues(__addable $left, __addable $right)
{
    return $left + $right;
}

processValues(
    new Money(5, 'USD'),
    new Vector2d(5, 10)
);
```
Despite both of the parameters to processValues implementing the _+addable interface, it's not trivial to check if it's possible to actually add the two types together.

Instead users are recommended to use specific types:

```
function processValues(Money $left, Money $right)
{
    return $left + $right;
}

processMoneyValues(
    new Money(5, 'USD'),
    new Vector2d(5, 10)
);

// Type error, Vector2d can't be used as Money
```

For more details please see [this](https://news-web.php.net/php.internals/115719) and [this](https://news-web.php.net/php.internals/115752).

### Why the identity operator is not overloadable

 The === operator is used to check whether two variables contain the same object, or whether two non-objects have the same type and value.

 The position of this RFC is that allowing the identity operator to be overloaded isn't a useful thing to do, which would only have bad uses.

### Why the RFC doesn't include logical operators

The logical operators &&, ||, and, or, and xor refer to a specific kind of math operation, boolean algebra, and their usage should reserved for that purpose only.

Most behavior that users would want to control with overloads to these operators can in fact be accomplished by allowing an object to control its casting to a boolean value. That is not part of this RFC, but the RFC author views that as a better way to address these operators than allowing arbitrary overloads.

## Backward Incompatible Changes

The sole known change is that objects used with operators will no longer result in a thrown TypeError, but instead a thrown InvalidOperator which extends TypeError/Error. TODO - decide

## Proposed PHP Version(s)
8.2

## RFC Impact

### To SAPIs
Describe the impact to CLI, Development web server, embedded PHP etc.

### To Existing Extensions

TODO - need to check existing things, like DateTime and GMP objects conform to rules here, otherwise might need to be changed.

### To Opcache

TODO

## Open Issues

## Unaffected PHP Functionality

## Future Scope

Someone else can propose touching the identity operator, or logic operators.

## Proposed Voting Choices
Accept the RFC and add user operator overloading to PHP

## Patches and Tests
The draft PR for this RFC can be found here: https://github.com/php/php-src/pull/7388

## Implementation
After the project is implemented, this section should contain
  - the version(s) it was merged into
  - a link to the git commit(s)
  - a link to the PHP manual entry for the feature
  - a link to the language specification section (if any)

## References
TODO

## Rejected Features

TODO