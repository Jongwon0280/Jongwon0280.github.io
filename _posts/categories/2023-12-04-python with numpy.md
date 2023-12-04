---
layout: single
title: Basics - Python with Numpy
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-04
---

## 넘파이를 활용한 신경망 기초

> **목적**
> 

뉴럴네트워크에 사용되는 벡터표현, 활성화함수등을 구현한다.

> **활용**
> 

Numpy라이브러리를 활용하여 구현한다.

> **시그모이드(Sigmoid)**
> 

- 공식 : 𝑠𝑖𝑔𝑚𝑜𝑖𝑑(𝑥)=1 / (1+e^-x)

![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/751a1c2a-6508-42f7-ac40-dc8db0904dc4)

로지스틱 함수로 알려진, 비선형함수를 구현한다. 뉴럴네트워크의 노드들의 활성화함수로 사용되었으며, 최근에는 이진분류의 출력층에서의 활성화함수로 주로 사용된다.

- math.exp()
    
    exp는 e^n을 의미한다.
    
    ```python
    import math
    
    def basic_sigmoid(x):
    	s = 1 / (1 + math.exp(-x))
    	return s
    
    print("basic_sigmoid(1) = " + str(basic_sigmoid(1)))
    
    #basic_sigmoid(1) = 0.7310585786300049
    
    t_x=np.array([1,2,3])
    
    print(np.exp(t_x))
    
    #[ 2.71828183  7.3890561  20.08553692]
    ```
    

- np.exp()
    
    벡터연산이 가능한 np.exp()로 math.exp()를 대체한다.
    
    ```python
    import numpy as np
    
    def sigmoid(x):
    	s = 1. / (1 + np.exp(-x))
    	return s
    
    t_x = np.array([1, 2, 3])
    print("sigmoid(t_x) = " + str(sigmoid(t_x)))
    
    sigmoid(t_x) = [0.73105858 0.88079708 0.95257413]
    ```
    

> **시그모이드 기울기**
> 

시그모이드의 미분은 𝜎′(𝑥)=𝜎(𝑥)(1−𝜎(𝑥))로 이루어진다.

```python
def sigmoid_derivative(x):
	s= sigmoid(x)
	ds = s * (1-s)
	return ds

t_x = np.array([1, 2, 3])
print ("sigmoid_derivative(t_x) = " + str(sigmoid_derivative(t_x)))
# sigmoid_derivative(t_x) = [0.19661193 0.10499359 0.04517666]
```

> **Reshape arrays**
> 

np.shape은 array의 shape을 얻을 수 있으며, reshape은 말그대로 변형할 수 있다.

![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/81bc6b74-9317-48d9-b112-7a1459fcf396)

(width, height, channel)로 구성되어있는 이미지를 벡터로 변형하기위해서 reshape과정이 필요하다.

- image2vector
    
    (width, height, 3) → (width*height*3,1)로 변형하여 NN의 입력으로 넣어줄 수 있게 변형한다.
    
    ```python
    def image2vector(image):
    	v= image.reshape((image.shape[0]*image.shape[1]*image.shape[2],1))
    	return v
    
    t_image = np.array([[[ 0.67826139,  0.29380381],
                         [ 0.90714982,  0.52835647],
                         [ 0.4215251 ,  0.45017551]],
    
                       [[ 0.92814219,  0.96677647],
                        [ 0.85304703,  0.52351845],
                        [ 0.19981397,  0.27417313]],
    
                       [[ 0.60659855,  0.00533165],
                        [ 0.10820313,  0.49978937],
                        [ 0.34144279,  0.94630077]]])
    
    print ("image2vector(image) = " + str(image2vector(t_image)))
    #image2vector(image) = [[0.67826139]
    # [0.29380381]
    # [0.90714982]
    # [0.52835647]
    # [0.4215251 ]
    # [0.45017551]
    # [0.92814219]
    # [0.96677647]
    # [0.85304703]
    # [0.52351845]
    # [0.19981397]
    # [0.27417313]
    # [0.60659855]
    # [0.00533165]
    # [0.10820313]
    # [0.49978937]
    # [0.34144279]
    # [0.94630077]]
    
    ```
    

> **Normalizing rows**
> 

ML과 DL에서의 정규화는 중요한 기술이다. 역전파연산을 빠르게 진행할 수 있다.

![Untitled 2](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/ce7e1f0b-b7f9-4b42-9d23-dd7b7b644db8)

- 행 정규화
    
    ```python
    def normalize_rows(x):
    	x_norm = np.linalg.norm(x,ord=2,axis=1,keepdims=True)
    	# ord는 L2 Norm을 수행하게 2로 지정
    	x= x / x_norm
    	return x
    
    x = np.array([[0, 3, 4],
                  [1, 6, 4]])
    print("normalizeRows(x) = " + str(normalize_rows(x)))
    #normalizeRows(x) = [[0.         0.6        0.8       ]
    # [0.13736056 0.82416338 0.54944226]]
    ```
    

> **소프트맥스(Softmax)**
> 

소프트맥스는 다중분류를 위한 출력층의 활성화함수이며, 자연로그를 통해 0-1 사이의 값으로 바꾸고, 이를 통해 각 클래스별 확률을 구한다.

![Untitled 3](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/e898f014-a546-4fab-84ec-6da32b4e523e)

```python
def softmax(x):
	x_exp = np.exp(x)
	x_sum = np.sum(x_exp,axis=1,keepdims=True)
	s = x_exp/x_sum
	return s 

t_x = np.array([[9, 2, 5, 0, 0],
                [7, 5, 0, 0 ,0]])
print("softmax(x) = " + str(softmax(t_x)))
#softmax(x) = [[9.80897665e-01 8.94462891e-04 1.79657674e-02 1.21052389e-04
#  1.21052389e-04]
# [8.78679856e-01 1.18916387e-01 8.01252314e-04 8.01252314e-04
#  8.01252314e-04]]
```

> **L1 and L2 손실 함수**
> 

- L1
    
    ![Untitled 4](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/92a596de-51ff-4c08-9ca5-6ecb8b9b0448)
    
    ```python
    def L1(yhat, y):
    	loss = np.sum(np.abs(y-yhat),axis=-1)
    	return loss
    
    yhat = np.array([.9, 0.2, 0.1, .4, .9])
    y = np.array([1, 0, 0, 1, 1])
    print("L1 = " + str(L1(yhat, y)))
    #L1 = 1.1
    ```
    

- L2
    
   
    ![Untitled 5](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/866b15f3-572d-4a04-b861-1ba4cdead298)

    np.dot을 통해 square를 구한다.
    
    ```python
    def L2(yhat,y):
    	loss= np.sum(np.dot(np.abs(y-yhat),np.abs(y-yhat)),-1)
    	return loss
    
    yhat = np.array([.9, 0.2, 0.1, .4, .9])
    y = np.array([1, 0, 0, 1, 1])
    
    print("L2 = " + str(L2(yhat, y)))
    L2 = 0.43
    ```
