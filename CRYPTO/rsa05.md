# task05 解析
题目如下
```python
#!/usr/bin/env python3

# from Crypto.Util.number import *

# from secret import FLAG

# BITS = 512

# p = getPrime(BITS)
# q = getPrime(BITS)
# N = p * q
# e = 65537
# d = inverse(e, (p - 1) * (q - 1))
# dp = d % (p - 1)

# pt = bytes_to_long(FLAG)

# ct = pow(pt, e, N)

# print((N, e))
# print(dp)
# print(ct)

```
可知
> d * e == 1 + k_1 * (p-1) * (q-1)
  d * e == dp * e + k_2 * (p-1) * e

两式相减得到
> dp * e - 1 == (k_1 * (q-1) - k_2 * e) * (p-1)

let $$i = k_1 * (q-1) - k_2 * e$$
then $$`dp * e - 1 == i * (p-1)`$$
`dp` is no more than `(p-1)`
so `i` is no more than `e`
we can find out `i` easily, and we have $$p = ((e * dp - 1) // i) + 1$$
