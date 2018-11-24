# [Golang初探] 之 strconv.Itoa

## 描述

**strconv.Itoa** 实现了数字（整数、浮点数等）与字符串之间的互相转换。

## Examples

```golang
package main

import (
    "fmt"
    "strconv"
)

func main() {

    // FormatUint
    v := uint64(42)

    s10 := strconv.FormatUint(v, 10)
    fmt.Printf("%T, %v\n", s10, s10)

    s16 := strconv.FormatUint(v, 16)
    fmt.Printf("%T, %v\n", s16, s16)

    /* Output:
        string, 42
        string, 2a
    */

    // FormatInt
    v := int64(-42)

    s10 := strconv.FormatInt(v, 10)
    fmt.Printf("%T, %v\n", s10, s10)

    s16 := strconv.FormatInt(v, 16)
    fmt.Printf("%T, %v\n", s16, s16)

    /* Output:
        string, -42
        string, -2a
    */

    // Itoa
    i := 10
    s := strconv.Itoa(i)
    fmt.Printf("%T, %v\n", s, s)
    /* Output:
        string, 10
    */

    // AppendInt
    b10 := []byte("int (base 10):")
    b10 = strconv.AppendInt(b10, -42, 10)
    fmt.Println(string(b10))

    b16 := []byte("int (base 16):")
    b16 = strconv.AppendInt(b16, -42, 16)
    fmt.Println(string(b16))
    /*
        int (base 10):-42
        int (base 16):-2a
    */

}
```

## 源码

```golang
/*
 * 函数原型
*/

const fastSmalls = true // 允许小整数快速转换

// FormatUint 返回已知进制 base(2 <= base <= 36) 的数值 i 的字符串表示
// 数字(digit values) >= 10 的时候，返回结果使用小写的字母 'a'-'z' 表示
func FormatUint(i uint64, base int) string {
    if fastSmalls && i < nSmalls && base == 10 {
        return small(int(i))
    }
    _, s := formatBits(nil, i, base, false, false)
    return s
}

// FormatInt 函数功能与 FormatUint 相同
// 不过 FormatInt 接收的整数参数是 int64 类型
// 而 FormatUint 接收的整数参数是 uint64 类型
func FormatInt(i int64, base int) string {
    if fastSmalls && 0 <= i && i < nSmalls && base == 10 {
        return small(int(i))
    }
    _, s := formatBits(nil, uint64(i), base, i < 0, false)
    return s
}

// Itoa 是函数调用 FormatInt(int64(i), 10) 的简写形式
func Itoa(i int) string {
    return FormatInt(int64(i), 10)
}

// AppendInt 执行类似于 FormatInt 的函数，将结果添加到 dst 字符流中
func AppendInt(dst []byte, i int64, base int) []byte {
    if fastSmalls && 0 <= i && i < nSmalls && base == 10 {
        return append(dst, small(int(i))...)
    }
    dst, _ = formatBits(dst, uint64(i), base, i < 0, true)
    return dst
}

// AppendInt 执行类似于 FormatUint 的函数，将结果添加到 dst 字符流中
func AppendUint(dst []byte, i uint64, base int) []byte {
    if fastSmalls && i < nSmalls && base == 10 {
        return append(dst, small(int(i))...)
    }
    dst, _ = formatBits(dst, i, base, false, true)
    return dst
}

/*
 * 小整数使用参考
*/

// [0, 100) 区间整数对应的字符串
func small(i int) string {
    if i < 10 {
        return digits[i : i+1]
    }
    return smallsString[i*2 : i*2+2]
}

const nSmalls = 100

const smallsString = "00010203040506070809" +
    "10111213141516171819" +
    "20212223242526272829" +
    "30313233343536373839" +
    "40414243444546474849" +
    "50515253545556575859" +
    "60616263646566676869" +
    "70717273747576777879" +
    "80818283848586878889" +
    "90919293949596979899"

const host32bit = ^uint(0)>>32 == 0

const digits = "0123456789abcdefghijklmnopqrstuvwxyz"

// formatBits 根据给定的进制数 base 计算 u 对应的字符串表示
// 如果 reg 为 true，表示 u 是 int64 位类型的负数
//
// 如果 append_ 为 true，则转化后的数值字符串追加到 dst 后面，
// 并且结果字节数组作为第一个返回值，否则结果作为第二个返回值
//
func formatBits(dst []byte, u uint64, base int, neg, append_ bool) (d []byte, s string) {

    // 不允许的进制位数
    if base < 2 || base > len(digits) {
        panic("strconv: illegal AppendInt/FormatInt base")
    }
    // 2 <==> 36 进制位

    var a [64 + 1]byte // +1 for sign of 64bit value in base 2
    i := len(a)

    // 如果为负数，则先取绝对值，u 为 uint64 类型
    if neg {
        u = -u
    }

    // convert bits
    // We use uint values where we can because those will
    // fit into a single register even on a 32bit machine.
    if base == 10 {
        // common case: use constants for / because
        // the compiler can optimize it into a multiply+shift

        if host32bit {
            // convert the lower digits using 32bit operations
            for u >= 1e9 {
                // Avoid using r = a%b in addition to q = a/b
                // since 64bit division and modulo operations
                // are calculated by runtime functions on 32bit machines.
                q := u / 1e9
                us := uint(u - q*1e9) // u % 1e9 fits into a uint
                for j := 4; j > 0; j-- {
                    is := us % 100 * 2
                    us /= 100
                    i -= 2
                    a[i+1] = smallsString[is+1]
                    a[i+0] = smallsString[is+0]
                }

                // us < 10, since it contains the last digit
                // from the initial 9-digit us.
                i--
                a[i] = smallsString[us*2+1]

                u = q
            }
            // u < 1e9
        }

        // u guaranteed to fit into a uint
        us := uint(u)
        for us >= 100 {
            is := us % 100 * 2
            us /= 100
            i -= 2
            a[i+1] = smallsString[is+1]
            a[i+0] = smallsString[is+0]
        }

        // us < 100
        is := us * 2
        i--
        a[i] = smallsString[is+1]
        if us >= 10 {
            i--
            a[i] = smallsString[is]
        }

    } else if isPowerOfTwo(base) {
        // It is known that base is a power of two and
        // 2 <= base <= len(digits).
        // Use shifts and masks instead of / and %.
        shift := uint(bits.TrailingZeros(uint(base))) & 31
        b := uint64(base)
        m := uint(base) - 1 // == 1<<shift - 1
        for u >= b {
            i--
            a[i] = digits[uint(u)&m]
            u >>= shift
        }
        // u < base
        i--
        a[i] = digits[uint(u)]
    } else {
        // general case
        b := uint64(base)
        for u >= b {
            i--
            // Avoid using r = a%b in addition to q = a/b
            // since 64bit division and modulo operations
            // are calculated by runtime functions on 32bit machines.
            q := u / b
            a[i] = digits[uint(u-q*b)]
            u = q
        }
        // u < base
        i--
        a[i] = digits[uint(u)]
    }

    // 如果需要，加上符号 "-"
    if neg {
        i--
        a[i] = '-'
    }

    if append_ {
        d = append(dst, a[i:]...)
        return
    }
    s = string(a[i:])
    return
}

func isPowerOfTwo(x int) bool {
    return x&(x-1) == 0
}

```
