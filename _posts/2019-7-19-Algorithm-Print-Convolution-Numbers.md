---
layout: post
title: Algorithm -- Print Convolution Numbers
---

问题：打印逆时针回旋数，例如：当n=5时，输出的回旋数如下所示：

![当n=5时，输出的回旋数][1]

## 算法一

### 思考方法

首先，从外到内把整个回旋数矩阵划分成一圈一圈的，然后，再按照从上到下、从左到右、从下到上、从右到左的顺序依次输出每圈的数字。

![算法一思路图][2]

### 代码实现

```java
public ConvolutionGenerator generate() {
    // 圈数；
    for (int r = 0; r < n / 2; r++) {
        // 从上到下；
        for (int i = r; i < n - 1 - r; i++) {
            convolutionNumbers[i][r] = number++;
        }

        // 从左到右；
        for (int j = r; j < n - 1 - r; j++) {
            convolutionNumbers[n - 1 - r][j] = number++;
        }

        // 从下到上；
        for (int k = n - 1 - r; k > r; k--) {
            convolutionNumbers[k][n - 1 - r] = number++;
        }

        // 从右到左；
        for (int l = n - 1 - r; l > r; l--) {
            convolutionNumbers[r][l] = number++;
        }
    }
    return this;
}
```

## 算法二

### 思考方法

首先，从外到内把整个回旋数矩阵划分成一圈一圈的，然后，再把每圈的数字分成左下半圈、右上半圈等2部分分别输出。

![算法二思路图][3]

### 代码实现

```java
public ConvolutionGenerator generate() {
    int a = 0;
    int b = n - 1 - a;
    while (b >= a) {
        // 左下半圈；
        for (int i = a; i <= b; i++) {
            for (int j = a; j <= b; j++) {
                if (i != b && j == a) {
                    convolutionNumbers[i][j] = number++;
                }
                if (i == b && j != b) {
                    convolutionNumbers[i][j] = number++;
                }
            }
        }

        // 右上半圈；
        for (int i = b; i >= a; i--) {
            for (int j = b; j >= a; j--) {
                if (i != a && j == b) {
                    convolutionNumbers[i][j] = number++;
                }
                if (i == a && j != a) {
                    convolutionNumbers[i][j] = number++;
                }
            }
        }

        a++;
        b = n - 1 - a;
    }
    return this;
}
```

## 测试

运行`new ConvolutionGenerator(5).generate().print();`，输出如下结果：

![测试结果][4]

如果要查看完整的代码，请点击[ConvolutionGenerator.java][5]！

[1]: ../images/2019/7/19/1.png
[2]: ../images/2019/7/19/2.png
[3]: ../images/2019/7/19/3.png
[4]: ../images/2019/7/19/4.png
[5]: https://github.com/Warnier-zhang/Notes/blob/master/src/main/java/org/warnier/zhang/notes/ConvolutionGenerator.java
