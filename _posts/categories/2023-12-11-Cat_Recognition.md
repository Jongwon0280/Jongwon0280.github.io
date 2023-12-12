---
layout: single
title: L layer NN - Cat Recognition
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-11
---

## L layer NN - Cat Recognition

> **목적**
> 

저번 과제는 Logistic Regression을 통해서 고양이 분류기를 구축하였지만, 이번엔 L레이어 NN을 통해 고양이 분류기를 구축한다.

> **데이터셋**
> 

고양이와 고양이가 아닌 이미지들로 구성되어있다.

![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/bdba3bf8-17b4-4e00-9685-a8302277de98)

```
Number of training examples: 209
Number of testing examples: 50
Each image is of size: (64, 64, 3)
train_x_orig shape: (209, 64, 64, 3)
train_y shape: (1, 209)
test_x_orig shape: (50, 64, 64, 3)
test_y shape: (1, 50)
```

> **모델 아키텍처**
> 

![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/88c7d976-08e2-4f52-8444-2cbfce69e33a)

```python
train_x_flatten = train_x_orig.reshape(train_x_orig.shape[0], -1).T   
test_x_flatten = test_x_orig.reshape(test_x_orig.shape[0], -1).T

train_x = train_x_flatten/255.
test_x = test_x_flatten/255.
```

![Untitled 2](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/00635c0f-b40c-41ef-b0ef-8368b5eedbfa)

Relu를 활성화함수로 사용하고, 출력층에서는 sigmoid를 사용하며, 손실함수로는 2진 크로스엔트로피를 사용한다. 이전 과제에서 사용한 함수들을 사용하여, 가중치초기화 , 전향파, 역전파, 최적화를 사용한다. 2층으로 hidden layer를 제한한다.

```python
n_x = 12288     # num_px * num_px * 3
n_h = 7
n_y = 1
layers_dims = (n_x, n_h, n_y)
learning_rate = 0.0075
```

```python
def two_layer_model(X, Y, layers_dims, learning_rate = 0.0075, num_iterations = 3000, print_cost=False):

	np.random.seed(1)
	grads = {}
	costs = []                              # to keep track of the cost
	m = X.shape[1]                           # number of examples
	(n_x, n_h, n_y) = layers_dims
	
	parameters=initialize_parameters(n_x,n_h,n_y)
	
	W1 = parameters["W1"]
	b1 = parameters["b1"]
	W2 = parameters["W2"]
	b2 = parameters["b2"]
	
	for i in tqdm(range(0, num_iterations)):
		A1,cache1 = linear_activation_forward(X, W1, b1, activation='relu')
	  A2,cache2 = linear_activation_forward(A1, W2, b2, activation='sigmoid')
		cost = compute_cost(A2,Y)
		
		# 역전파
		dA2 = - (np.divide(Y, A2) - np.divide(1 - Y, 1 - A2))
		dA1, dW2, db2=linear_activation_backward(dA2, cache2, activation='sigmoid')
	  dA0, dW1, db1=linear_activation_backward(dA1, cache1, activation='relu')
		
		grads['dW1'] = dW1
	  grads['db1'] = db1
	  grads['dW2'] = dW2
	  grads['db2'] = db2
	
	
		W1 = parameters["W1"]
	  b1 = parameters["b1"]
	  W2 = parameters["W2"]
	  b2 = parameters["b2"]
		
		# 배치사이즈는 100이다. 3000 / 100 = 30이므로 에폭은 30
		if print_cost and i % 100 == 0 or i == num_iterations - 1:
	        print("Cost after iteration {}: {}".format(i, np.squeeze(cost)))
	  if i % 100 == 0 or i == num_iterations:
	        costs.append(cost)
	
	return parameters, costs

def plot_costs(costs, learning_rate=0.0075):
    plt.plot(np.squeeze(costs))
    plt.ylabel('cost')
    plt.xlabel('iterations (per hundreds)')
    plt.title("Learning rate =" + str(learning_rate))
    plt.show()

parameters, costs = two_layer_model(train_x, train_y, layers_dims = (n_x, n_h, n_y), num_iterations = 2500, print_cost=True)
plot_costs(costs, learning_rate)

# train acc : 0.99 / test acc : 0.72
```

> **L-layer NN**
> 

여기서도 전에 구현한 L-layer NN에서 구현한 함수를 사용한다.

```python
layers_dims = [12288, 20, 7, 5, 1] # 각 층별 노드수이다.
```

```python

from tqdm import tqdm
def L_layer_model(X, Y, layers_dims, learning_rate = 0.0075, num_iterations = 3000, print_cost=False):
    

    np.random.seed(1)
    costs = []                         # keep track of cost
    
   
    parameters = initialize_parameters_deep(layers_dims)
   
    for i in tqdm(range(0, num_iterations)):

        
        AL, caches=L_model_forward(X, parameters)
        
        cost = compute_cost(AL, Y)
        
        grads = L_model_backward(AL, Y, caches)
        
        parameters = update_parameters(parameters, grads, learning_rate)
        
        
        if print_cost and i % 100 == 0 or i == num_iterations - 1:
            print("Cost after iteration {}: {}".format(i, np.squeeze(cost)))
        if i % 100 == 0 or i == num_iterations:
            costs.append(cost)
    
    return parameters, cost

parameters, costs = L_layer_model(train_x, train_y, layers_dims, num_iterations = 2500, print_cost = True)

# train acc : 0.99 / test acc : 0.8

```

> **결과**
> 

| train/test | 2Layer | 4Layer |
| --- | --- | --- |
| train | 0.99 | 0.99 |
| test | 0.72 | 0.8 |

레이어를 증가 시켰을때 더 높은 정확도를 얻을 수 있다.

하지만, NN으로 80%의 성능은 만족스럽지 못하다.