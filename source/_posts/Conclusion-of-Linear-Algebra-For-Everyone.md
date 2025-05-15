---
title: Conclusion of Linear Algebra For Everyone
date: 2025-05-14 19:40:56
tags:
  - mathmatic
  - linear algebra
---

# Conclusion of **Linear Algebra For Everyone**

## REFERENCES

- [The-Art-of-Linear-Algebra](https://github.com/kenjihiranabe/The-Art-of-Linear-Algebra)

## Glossary of Terms

|           terms           |                         descprition                          |
| :-----------------------: | :----------------------------------------------------------: |
|         `Matrix`          | A rectangular array of numbers, symbols, or expressions arranged in rows and columns. |
|         `Vector`          | An ordered set of numbers, often represented as a column or row matrix. |
|         `Scalar`          |      A single number, as opposed to a vector or matrix.      |
|        `Transpose`        |      A matrix operation that flips a matrix over its .       |
|       `Determinant`       |   A scalar value that can be computed from a square matrix   |
|       `Eigenvector`       | A non-zero vector that only changes by a scalar factor when a linear transformation is applied to it. |
|       `Eigenvalue`        | The scalar factor by which an eigenvector is scaled during a linear transformation. |
|          `Rank`           | The dimension of the column space or row space of a matrix.  |
|       `Dot Product`       | The scalar product of two vectors, obtained by multiplying the corresponding components and summing the results. |
|      `Cross Product`      | The vector product of two vectors in three-dimensional space, resulting in a vector that is  to the original vectors |
|       `Orthogonal`        |   Two vectors are orthogonal if their dot product is zero    |
|          `space`          |  Represents a collection of vectors with certain properties  |
|      `Vector Space`       | A vector space is a set of vectors along with operations of vector addition and scalar multiplication that satisfy certain properties. |
|        `Subspace`         | A subspace is a subset of a vector space that is itself a vector space. It contains the zero vector, is closed under vector addition and scalar multiplication, and satisfies the other properties of a vector space. |
|   `Inner product space`   | An inner product space is a vector space equipped with an inner product, which is a generalization of the dot product. The inner product defines a notion of angle and length in the space. |
|   `Normed Vector Space`   | A normed vector space is a vector space equipped with a norm, which is a function that assigns a non-negative length to each vector and satisfies certain properties. |
|      `Hilbert Space`      | A Hilbert space is a complete inner product space, meaning that it is a vector space equipped with an inner product that is also a complete metric space with respect to the norm induced by the inner product. |
| `Upper Triangular Matrix` | A Upper Triangular Matrix is a matrix in which all the elements below the main diagonal are zero, and all the elements on or above the main diagonal can be zero or non-zero. |
| `Lower Triangular Matrix` | A Lower Triangular Matrix is a matrix in which all the elements above the main diagonal (the diagonal from the top left to the bottom right) are zero, and all the elements on or below the main diagonal can be zero or non-zero. |
|    `Orthogonal Matrix`    |                                                              |
|           `CR`            |            C stands for Column, R stands for Row             |
|           `LU`            |            Lower tirangular and Upper triangular             |
|           `QR`            | QR decomposition as Gram-Schmidt orthogonalization Orthogonal Q and triangular R |

## Summary

1. A scarlar times a row vector produces a row vector;
2. A scarlar times a column vector produces a column vector;
3. Multiplying a row vector by a column vector results in a scalar value, which is obtained by summing the products of the corresponding elements of the two vectors;
4. Multipling a column vector with a row vector produces a matrix. Assuming the height of column vector is `M` and the length of row vector is `N`, the matrix will be a Rank 1 matrix with dimensions **M * N**;
5. According the rules `3` and `4`, we suprisingly find that **a row vector times a column vector produces a scalar value, while a column vector times a row vector produces a matrix.**, here is the explanation:
     1. **Row Vector times Column Vector (Dot Product):**  When a row vector (1 x n matrix) is multiplied by a column vector (n x 1 matrix), the resulting product is a scalar value, known as the dot product. This is because the dimensions of the row vector and column vector are such that the inner dimensions match (n), allowing for the dot product operation which results in a scalar.
     2. **Column Vector times Row Vector (Outer Product):** When a column vector (m x 1 matrix) is multiplied by a row vector (1 x n matrix), the resulting product is a matrix, known as the outer product or tensor product. The dimensions of the column vector and row vector are such that the outer product operation results in a matrix with dimensions m x n, where each element of the resulting matrix is obtained by multiplying the corresponding elements of the column vector and row vector.
6. Lower triangular matrix times upper triangular matrix leads to Gaussian elimination, thus resulting in a matrix filled with zero except the diagnal‌ line.

## Content

### 2. Vector times Vector - 2 Ways

- **Dot product** ($\vec{a} \cdot \vec{b}$) is expressed as $\vec{a} ^T \vec{b}$ in matrix language and yields a `number`
- **Cross Product** $\vec{a} \times \vec{b}$ : $\vec{a} \vec{b}^T$ is a matrix. If neither a, b are 0, the result A is a rank 1 matrix.

#### Dot Product

$$
\begin{bmatrix}
1 & 2 & 3
\end{bmatrix}
\begin{bmatrix}
x_1 \\
x_2 \\
x_3
\end{bmatrix}  =
\begin{bmatrix}
1 \\
2 \\
3
\end{bmatrix}
\cdot
\begin{bmatrix}
x_1 \\
x_2 \\
x_3
\end{bmatrix} = x_1 + 2x_2 + 3x_3
$$

#### Cross Product

$$
\begin{bmatrix}
1 \\
2 \\
3
\end{bmatrix}
\begin{bmatrix}
x & y
\end{bmatrix} =
\begin{bmatrix}
1 \\
2 \\
3
\end{bmatrix}
\times
\begin{bmatrix}
x \\
y
\end{bmatrix} =
\begin{bmatrix}
x & y \\
2x & 2y \\
3x & 3y
\end{bmatrix}
$$



### 3. Matrix times Vector - 2 Ways

A matrix times a vector creates a vecotr of three **dot product** as well as a linear combination of the column vectors of A.

> The row vectors of A are multiplied by a vector $x$ and become the three dot-product elements of $Ax$

$$
Ax = \begin{bmatrix}
1 & 2 \\
3 & 4 \\
5 & 6
\end{bmatrix}
\begin{bmatrix}
x_1 \\
x_2
\end{bmatrix} =
\begin{bmatrix}
(x_1 + 2x_2) \\
(3x_1 + 4x_2) \\
(5x_1 + 6x_2)
\end{bmatrix}
$$

> The product $Ax$ is a linear combination of the column vectors of A.

$$
Ax = \begin{bmatrix}
1 & 2 \\
3 & 4 \\
5 & 6
\end{bmatrix}
\begin{bmatrix}
x_1 \\
x_2
\end{bmatrix} =
x_1\begin{bmatrix}
1 \\
3 \\
5
\end{bmatrix} +
x_2\begin{bmatrix}
2 \\
4 \\
6
\end{bmatrix}
$$

Also, (vM1) and (vM2) show the same pattern of a row vector times a matrix

> A row vector `y` multiplied by the two column vectors of A and become the two dot-product elements of $yA$

$$
yA =
\begin{bmatrix}
y_1 & y_2 & y_3
\end{bmatrix}
\begin{bmatrix}
1 & 2 \\
3 & 4 \\
5 & 6
\end{bmatrix} =
\begin{bmatrix}
(y_1 + 3y_2 + 5y_3) & (2y_1 + 4y_2 + 6y_3)
\end{bmatrix}
$$

> The product $yA$ is a linear combination of the row vectors of A.

$$
yA =
\begin{bmatrix}
y_1 & y_2 & y_3
\end{bmatrix}
\begin{bmatrix}
1 & 2 \\
3 & 4 \\
5 & 6
\end{bmatrix} =
y_1
\begin{bmatrix}
1 & 2
\end{bmatrix} +
y_2
\begin{bmatrix}
3 & 4
\end{bmatrix} +
y_3
\begin{bmatrix}
5 & 6
\end{bmatrix}
$$

### 4.  Matrix times Matrix - 4 ways

> A matrix with `M` rows and `N` columns times another matrix with `N` rows and `L` columns produces a matrix of `M` rows and `L` columns, each element is a **dot product**.
>
> Assuming we have two matrices, denoted as `A` and `B`, where A is an **3 \* 2** matrix and B is an **2\* 2** matrix. Multiplying the two matrices produces a new matrix `C`, and its value can be computed by the following formula:
>
> $C(x, y) = A(x, 0) * B(0, y) + A(x, 1) * B(1, y)$

$$
\begin{bmatrix}
1 & 2 \\
3 & 4 \\
5 & 6
\end{bmatrix}
\begin{bmatrix}
x_1 & y_1 \\
x_2 & y_2
\end{bmatrix} =
\begin{bmatrix}
(x_1 + 2x_2) & (y_1 + 2y_2) \\
(3x_1 + 4x_2) & (3y_1 + 4y_2) \\
(5x_1 + 6x_2) & (5y_1 + 5y_2)
\end{bmatrix}
$$

> $Ax$ and $Ay$ are linear combinations of columns of $A$. View the matrix on the left as a whole and view each row in matrix on the right as a whole,  $\begin{bmatrix}A\end{bmatrix}\begin{bmatrix}x & y\end{bmatrix}$ can be transformed into $\begin{bmatrix}Ax & Ay\end{bmatrix}$.
>
> In the following example, the formulate is transformed into the linear combinations of a **3 x 2** matrix times a **2 * 1** matrix which produces a **3 * 1** new matrix.

$$
\begin{bmatrix}
1 & 2 \\
3 & 4 \\
5 & 6
\end{bmatrix}
\begin{bmatrix}
x_1 & y_1 \\
x_2 & y_2
\end{bmatrix} =
\begin{bmatrix}
A
\end{bmatrix}
\begin{bmatrix}
x & y
\end{bmatrix} =
\begin{bmatrix}
Ax & Ay
\end{bmatrix}
$$

> The produced rows are linear combinations of rows. 
>
> View each column in the matrix on the left as a whole and view the matrix on the right as a whole. The following matrix can be transformed into a matrix where every row is a **1 \* 2** matrix produced from a **1 \* 2** matrix multiplied by a **2 \* 2** matrix

$$
\begin{bmatrix}
1 & 2 \\
3 & 4 \\
5 & 6
\end{bmatrix}
 \begin{bmatrix}
x_1 & y_1 \\
x_2 & y_2
\end{bmatrix} = 
\begin{bmatrix}
a_1^* \\
a_2^* \\
a_3^*
\end{bmatrix} X =
\begin{bmatrix}
a_1^*X \\
a_2^*X \\
a_3^*X
\end{bmatrix}
$$

> View each column in matrix on the left as a whole and view each row in the matrix on the right as a whole, multiplication $AB$ is broken down to a sum of **rank 1 matrix**.

$$
\begin{bmatrix}
1 & 2 \\
3 & 4 \\
5 & 6
\end{bmatrix}
\begin{bmatrix}
x_1 & y_1 \\
x_2 & y_2
\end{bmatrix} =
\begin{bmatrix}
a_1 & a2
\end{bmatrix}
\begin{bmatrix}
b_1^* \\
b_2^*
\end{bmatrix} =
\begin{bmatrix}
a_1b_1^* \\
a_2b_2^*
\end{bmatrix} =
\begin{bmatrix}
1 \\
3 \\
5
\end{bmatrix}
\begin{bmatrix}
b_{11} & b_{12}
\end{bmatrix} +
\begin{bmatrix}
2 \\
4 \\
6
\end{bmatrix}
\begin{bmatrix}
b_{21} & b_{22}
\end{bmatrix}
$$



