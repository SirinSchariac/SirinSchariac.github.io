---
layout: 	post  				   
title: 		UMich DL for CV   				
subtitle:	Neural Network
date:       2024-12-12 				
author:     Sirin 						
header-img: img/universe.jpg
catalog: 	true 				
tags:						
    - Lecture
    - CV
    - DL
---

### Neural Network

#### Feature Transformation

for example, the original space is a Cartesian coordinate system. After mathematic transformation, we can turn it into Polar coordinate system, called feature space. 

In this situation, a nonlinear classifier in the original space can become a linear classifier in the feature space, which is more convenient to implement. 

An application is Histogram of Oriented Gradients (HoG), the method:

1. Compute edge direction/strength at each pixel
2. Divide image into 8*8 regions
3. Within each region, compute a histogram of edge directions weighted by the edge strength.

#### Fully-connected Neural Network

![Lec5-FullyConnected.png](https://s2.loli.net/2024/12/03/FYgwk29n1y4bjRB.png)

the `max` function in `f` is called the **activation function**, a classical implementation is **ReLU:**

```python
#Rectified Linear Unit
def ReLU(z):
	return max(0, z)
```

> Q: What if we build a neural network without activation function? 
>
> A: In this situation f = W_2 * W_1 * x, assuming W_3 = W_2 * W_1, we get f = W_3 * x. This is still a linear classifier!

#### Why ReLU can work?

![Lec5-SpaceWarping.png](https://s2.loli.net/2024/12/03/6imrcMsbhvF8N1o.png)

![Lec5-DataClouds.png](https://s2.loli.net/2024/12/03/QtYvsr9pIhwJM86.png)

The two pictures above show that ReLU can transform the non-linear boundary in original space to linear boundary in feature space.

### Back Propagation

> an example on back propagation

以$f=(x+y)z$为例，该算式的前向传播过程可以表达为$q=x+y\quad f=qz$，反向传播的目的是为了计算偏差$\frac{\part f}{\part x}$，$\frac{\part f}{\part y}$和$\frac{\part f}{\part z}$。根据链式法则可知，$\frac{\part f}{\part x}=\frac{\part f}{\part q}\frac{\part q}{\part x}$，其中，从$q=x+y$这一步运算角度来看，$\frac{\part f}{\part x}$称为*Downstream Gradient*，而$\frac{\part q}{\part x}$称为*Local Gradient*，$\frac{\part f}{\part q}$则称为*Upstream Gradient*。

> Sigmoid Function 

$\sigma(x)=\large \frac{1}{1+e^{-x}}$

$\frac{\part \sigma(x)}{\part x}=(1-\sigma(x))\sigma(x)$

> Pattern Gradient

![lec6-pattern.png](https://s2.loli.net/2024/12/17/aW7fQCn1BzevqxN.png)

> Matrix Operation

![lec6-MatrixGradient.png](https://s2.loli.net/2024/12/18/rfRTYLMdmzu2AlE.png)

在真正的神经网络中，输入和输出一般是以矩阵的形式给出，这种情况下的对于$\frac{\part L}{\part x}$的计算就如上图所示。由于矩阵的维度D，M都很大，直接计算出所有的梯度是不可能的(out of memory)，因此，需要采用切片的方式，每次计算一部分的梯度值。例如，先计算$\frac{\part L}{\part x_{1,1}}$，那么相当于要计算$\frac{\part L}{\part y}\frac{\part y}{\part x_{1,1}}$，$\frac{\part L}{\part y}$作为upstream gradient是已知的，$\frac{\part y}{\part x_{1,1}}$形状应该与$y$相同，只需要依次求出$\frac{\part y_{i,j}}{\part x_{1,1}}$的值即可，例如，图中就给出了$\frac{\part y_{1,2}}{\part x_{1,1}}$的计算方法，考虑$\frac{\part y_{2,3}}{\part x_{1,1}}$，由于$y_{2,3}=x_{2,1}w_{1,3}+x_{2,2}w_{2,3}+x_{2,3}w_{3,3}$，所以$\frac{\part y_{2,3}}{\part x_{1,1}}=0$。

从上述的推导中，就可以总结出一个规律：

$\LARGE \frac{\part L}{\part x_{i,j}}=\frac{\part L}{\part y}\frac{\part y}{\part x_{i,j}}=(w_{j,:})\cdot (\frac{\part L}{\part y_{i,:}})$

