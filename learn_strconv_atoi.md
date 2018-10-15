# [Golang] 初探之 strconv.Atoi

## 描述

**strconv** 包实现了字符串与数字（整数、浮点数等）之间的互相转换。

## Examples

```golang
package main

import (
	"fmt"
	"strconv"
)

func main() {

    // ParseUint
	v := "42"
	if s, err := strconv.ParseUint(v, 10, 32); err == nil {
		fmt.Printf("%T, %v\n", s, s)
	}
	if s, err := strconv.ParseUint(v, 10, 64); err == nil {
		fmt.Printf("%T, %v\n", s, s)
    }
    
    /* Output:
        uint64, 42
        uint64, 42
    */

    // ParseInt
    v32 := "-354634382"
	if s, err := strconv.ParseInt(v32, 10, 32); err == nil {
		fmt.Printf("%T, %v\n", s, s)
	}
	if s, err := strconv.ParseInt(v32, 16, 32); err == nil {
		fmt.Printf("%T, %v\n", s, s)
	}

	v64 := "-3546343826724305832"
	if s, err := strconv.ParseInt(v64, 10, 64); err == nil {
		fmt.Printf("%T, %v\n", s, s)
	}
	if s, err := strconv.ParseInt(v64, 16, 64); err == nil {
		fmt.Printf("%T, %v\n", s, s)
    }
    
    /* Output:
        int64, -354634382
        int64, -3546343826724305832
    */

    // Atoi
    v := "10"
	if s, err := strconv.Atoi(v); err == nil {
		fmt.Printf("%T, %v", s, s)
    }
    /* Output:
        int, 10
    */

}
```


## 源码


```golang

// uint 位数
const intSize = 32 << (^uint(0) >> 63)

// IntSize is the size in bits of an int or uint value.
const IntSize = intSize

// uint64 最大数值
const maxUint64 = (1<<64 - 1)

// 无符号整型
// @params s: 待转化的数字字符串
// @params base: 进制
// @params bitSize: 位数
// ParseUint is like ParseInt but for unsigned numbers.
func ParseUint(s string, base int, bitSize int) (uint64, error) {
    const fnParseUint = "ParseUint"

    if len(s) == 0 {
        return 0, syntaxError(fnParseUint, s)
    }

    s0 := s
    switch {
    case 2 <= base && base <= 36:
        // 允许传入的进制数

    case base == 0:
        // 根据字符串前缀确认进制
        switch {
        case s[0] == '0' && len(s) > 1 && (s[1] == 'x' || s[1] == 'X'):
            // 0x123 或 0X123 为16进制
            if len(s) < 3 {
                return 0, syntaxError(fnParseUint, s0)
            }
            base = 16
            s = s[2:]
        case s[0] == '0':
            // 0123 为8进制
            base = 8
            s = s[1:]
        default:
            // 其他为10进制
            base = 10
        }

    default:
        // 错误的进制数
        return 0, baseError(fnParseUint, s0, base)
    }

    // 位数默认0 为系统判定
    if bitSize == 0 {
        bitSize = int(IntSize)
    } else if bitSize < 0 || bitSize > 64 {
        return 0, bitSizeError(fnParseUint, s0, bitSize)
    }

    // Cutoff is the smallest number such that cutoff*base > maxUint64.
    // Use compile-time constants for common cases.
    var cutoff uint64
    switch base {
    case 10:
        cutoff = maxUint64/10 + 1
    case 16:
        cutoff = maxUint64/16 + 1
    default:
        cutoff = maxUint64/uint64(base) + 1
    }

    // 指定位数 bitSite 内允许的最大数值
    maxVal := uint64(1)<<uint(bitSize) - 1

    var n uint64
    for _, c := range []byte(s) {
        var d byte
        switch {
        case '0' <= c && c <= '9':
            d = c - '0'
        case 'a' <= c && c <= 'z':
            d = c - 'a' + 10
        case 'A' <= c && c <= 'Z':
            d = c - 'A' + 10
        default:
            return 0, syntaxError(fnParseUint, s0)
        }
        // 判断孰否超出进制范围
        if d >= byte(base) {
            return 0, syntaxError(fnParseUint, s0)
        }

        // 判断下一步后是否越界
        if n >= cutoff {
            // n*base overflows
            return maxVal, rangeError(fnParseUint, s0)
        }
        // 进位&加值
        n *= uint64(base)
        n1 := n + uint64(d)

        // 判断是否越界
        if n1 < n || n1 > maxVal {
            // n+v overflows
            return maxVal, rangeError(fnParseUint, s0)
        }
        n = n1
    }

    return n, nil
}


// ParseInt 根据给定的进制(0, 2 to 36)和位数大小(0 to 64)将字符串转化后返回相应的值
//
// if base == 0:
//      prefix -> "0x"; base = 16
//      prefix -> "0" : base = 8
// if base <2 || base > 36:
//      return err  
//
// 参数 bitSize 给出转换结果所允许的大小
//      0 -> int
//      8 -> int8
//      16 -> int16
//      32 -> int32
//      64 -> int64
//      x < 0 || x >64 -> err
//
func ParseInt(s string, base int, bitSize int) (i int64, err error) {
    const fnParseInt = "ParseInt"

    // Empty string bad.
    if len(s) == 0 {
        return 0, syntaxError(fnParseInt, s)
    }

    // 判断正负
    s0 := s
    neg := false
    if s[0] == '+' {
        s = s[1:]
    } else if s[0] == '-' {
        neg = true
        s = s[1:]
    }

    // 转换 uint 并且检查溢出.
    var un uint64
    un, err = ParseUint(s, base, bitSize)
    if err != nil && err.(*NumError).Err != ErrRange {
        err.(*NumError).Func = fnParseInt
        err.(*NumError).Num = s0
        return 0, err
    }

    if bitSize == 0 {
        bitSize = int(IntSize)
    }

    // 检查正负数是否 // +溢出
    cutoff := uint64(1 << uint(bitSize-1))
    if !neg && un >= cutoff { // +
        return int64(cutoff - 1), rangeError(fnParseInt, s0)
    }
    if neg && un > cutoff { // -
        return -int64(cutoff), rangeError(fnParseInt, s0)
    }
    n := int64(un)
    if neg {
        n = -n
    }
    return n, nil
}

// Atoi 和调用 ParseInt(s, 10, 0) 形同，只是返回 int 类型
// Atoi returns the result of ParseInt(s, 10, 0) converted to type int.
func Atoi(s string) (int, error) {
    const fnAtoi = "Atoi"

    sLen := len(s)
    if intSize == 32 && (0 < sLen && sLen < 10) ||
        intSize == 64 && (0 < sLen && sLen < 19) {
        // 快速转化路径：适合int类型的小整数.
        s0 := s
        if s[0] == '-' || s[0] == '+' {
            s = s[1:]
            if len(s) < 1 {
                return 0, &NumError{fnAtoi, s0, ErrSyntax}
            }
        }

        n := 0
        for _, ch := range []byte(s) {
            ch -= '0'
            if ch > 9 {
                return 0, &NumError{fnAtoi, s0, ErrSyntax}
            }
            n = n*10 + int(ch)
        }
        if s0[0] == '-' {
            n = -n
        }
        return n, nil
    }

    // 大整数.
    i64, err := ParseInt(s, 10, 0)
    if nerr, ok := err.(*NumError); ok {
        nerr.Func = fnAtoi
    }
    return int(i64), err
}

```




