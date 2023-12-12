---
layout: single
title: Improving NN - Regularization
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-11
---

## Improving NN - Regularization

> **목적**
> 

정규화를 NN에 적용하기

> **데이터셋**
> 

![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/031ce23a-8b8d-443f-97ed-4edf06609406)

![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/1bf425b9-2bf4-4511-8294-2032d0388215)

> **모델**
> 

```python
def model(X, Y, learning_rate = 0.3, num_iterations = 30000, print_cost = True, lambd = 0, keep_prob = 1):
    """
    Implements a three-layer neural network: LINEAR->RELU->LINEAR->RELU->LINEAR->SIGMOID.
    
    Arguments:
    X -- input data, of shape (input size, number of examples)
    Y -- true "label" vector (1 for blue dot / 0 for red dot), of shape (output size, number of examples)
    learning_rate -- learning rate of the optimization
    num_iterations -- number of iterations of the optimization loop
    print_cost -- If True, print the cost every 10000 iterations
    lambd -- regularization hyperparameter, scalar
    keep_prob - probability of keeping a neuron active during drop-out, scalar.
    
    Returns:
    parameters -- parameters learned by the model. They can then be used to predict.
    """
        
    grads = {}
    costs = []                            # to keep track of the cost
    m = X.shape[1]                        # number of examples
    layers_dims = [X.shape[0], 20, 3, 1]
    
    # Initialize parameters dictionary.
    parameters = initialize_parameters(layers_dims)

    # Loop (gradient descent)

    for i in range(0, num_iterations):

        # Forward propagation: LINEAR -> RELU -> LINEAR -> RELU -> LINEAR -> SIGMOID.
        if keep_prob == 1:
            a3, cache = forward_propagation(X, parameters)
        elif keep_prob < 1:
            a3, cache = forward_propagation_with_dropout(X, parameters, keep_prob)
        
        # Cost function
        if lambd == 0:
            cost = compute_cost(a3, Y)
        else:
            cost = compute_cost_with_regularization(a3, Y, parameters, lambd)
            
        # Backward propagation.
        assert (lambd == 0 or keep_prob == 1)   # it is possible to use both L2 regularization and dropout, 
                                                # but this assignment will only explore one at a time
        if lambd == 0 and keep_prob == 1:
            grads = backward_propagation(X, Y, cache)
        elif lambd != 0:
            grads = backward_propagation_with_regularization(X, Y, cache, lambd)
        elif keep_prob < 1:
            grads = backward_propagation_with_dropout(X, Y, cache, keep_prob)
        
        # Update parameters.
        parameters = update_parameters(parameters, grads, learning_rate)
        
        # Print the loss every 10000 iterations
        if print_cost and i % 10000 == 0:
            print("Cost after iteration {}: {}".format(i, cost))
        if print_cost and i % 1000 == 0:
            costs.append(cost)
    
    # plot the cost
    plt.plot(costs)
    plt.ylabel('cost')
    plt.xlabel('iterations (x1,000)')
    plt.title("Learning rate =" + str(learning_rate))
    plt.show()
    
    return parameters
```

> **정규화**
> 

- **비정규화 시 성능확인**
    
    ```python
    parameters = model(train_X, train_Y)
    print ("On the training set:")
    predictions_train = predict(train_X, train_Y, parameters)
    print ("On the test set:")
    predictions_test = predict(test_X, test_Y, parameters)
    ```
    
    ```
    Cost after iteration 0: 0.6557412523481002
    Cost after iteration 10000: 0.16329987525724204
    Cost after iteration 20000: 0.13851642423234922
    
    On the training set:
    Accuracy: 0.9478672985781991
    On the test set:
    Accuracy: 0.915
    ```
    
    ![Untitled 2](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/4e056bf5-46ff-4403-8955-8daa22d97d40)
    
    ![Untitled 3](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/e9bc549f-1818-4c42-9e13-d8b05f4156f7)
    
    비정규화시에 overfitting되는것을 확인할 수 있다.
    
- **L2 정규화**
    
    ![Untitled 4](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/c10057e1-cea3-4f70-b8ba-85b8001b38f6)
    
    ```python
    def compute_cost_with_regularization(A3, Y, parameters, lambd):
       
        m = Y.shape[1]
        W1 = parameters["W1"]
        W2 = parameters["W2"]
        W3 = parameters["W3"]
        
        cross_entropy_cost = compute_cost(A3, Y) 
        
        L2_regularization_cost=(1/m) * (lambd/2) * (np.sum(np.square(W1))+np.sum(np.square(W2))+np.sum(np.square(W3)))
        
        
        cost = cross_entropy_cost + L2_regularization_cost
        
        return cost
    ```
    
- **L2 정규화 역전파 구현**
    
    
    위에 정규화된 함수로 계산된 손실함수를 미분하면 덧셈으로 연결된 정규화 텀은 각 레이어의 가중치에 대한 편미분시 원래 가중치에 lambd/m을 곱해서 더해주면 되고, 이는 가중치값에 패널티읕 부여함으로써, 극단적으로 큰 lambd값은 모델이 선형성만을 학습할 수 있게 할수도 있다
    
     
    
    ```python
    def backward_propagation_with_regularization(X, Y, cache, lambd):
       
        
        m = X.shape[1]
        (Z1, A1, W1, b1, Z2, A2, W2, b2, Z3, A3, W3, b3) = cache
        
        dZ3 = A3 - Y
        
        dW3 = 1./m * np.dot(dZ3, A2.T) + ((lambd/m) * W3)
        
       
        db3 = 1. / m * np.sum(dZ3, axis=1, keepdims=True)
        
        dA2 = np.dot(W3.T, dZ3)
        dZ2 = np.multiply(dA2, np.int64(A2 > 0))
       
        dW2 = 1./m * np.dot(dZ2, A1.T) + ((lambd/m) * W2)
        
        db2 = 1. / m * np.sum(dZ2, axis=1, keepdims=True)
        
        dA1 = np.dot(W2.T, dZ2)
        dZ1 = np.multiply(dA1, np.int64(A1 > 0))
       
        dW1 = 1./m * np.dot(dZ1, X.T) + ((lambd/m) * W1)
        
        db1 = 1. / m * np.sum(dZ1, axis=1, keepdims=True)
        
        gradients = {"dZ3": dZ3, "dW3": dW3, "db3": db3,"dA2": dA2,
                     "dZ2": dZ2, "dW2": dW2, "db2": db2, "dA1": dA1, 
                     "dZ1": dZ1, "dW1": dW1, "db1": db1}
        
        return gradients
    ```
    
    ```python
    parameters = model(train_X, train_Y, lambd = 0.7)
    print ("On the train set:")
    predictions_train = predict(train_X, train_Y, parameters)
    print ("On the test set:")
    predictions_test = predict(test_X, test_Y, parameters)
    ```
    
    ```
    Cost after iteration 0: 0.6974484493131264
    Cost after iteration 10000: 0.2684918873282238
    Cost after iteration 20000: 0.26809163371273004
    
    On the train set:
    Accuracy: 0.9383886255924171
    On the test set:
    Accuracy: 0.93
    ```
    
    ![Untitled 5](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/4ebf3451-1a29-409b-9d2f-656ae29e30ee)
    
    ![Untitled 6](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/00eea541-a780-4a56-a713-106c6177b16c)
    
    lambd 파라미터를 조정하며, bias와 variance간의 적절한 지점을 찾는 것이중요하다. 너무 높은 lambd 값은 모델이 너무 지나친 bias를 갖게하며, 과소적합으로 이어지고, 너무 작은 lambd는 과적합으로 이어질 수 있기 때문이다.
    

> **Dropout**
> 

각 레이어에서 뉴런의 연결을 끊음으로써, 정규화역할을 할 수 있으며, 각 레이어별로, keep_prob 파라미터를 통해 얼마의 뉴런을 남길지 정할 수 있다.

- **Forwarding with Dropout**
    
    
    각 레이어 별로, keep_prob에 맞춰, random.rand를 통해 균일분포에서 랜덤값을 생성하여, 마스크 형태로 만들고, 각 뉴런의 활성화함수 결과값에 multiply하여, 유지할 비율에 맞게 활성화값을 보낸다. 여기서, 기존 뉴런들의 기댓값을 충족시키기 위해 **inverted dropout**을 시행한다.
    
    ```python
    def forward_propagation_with_dropout(X, parameters, keep_prob = 0.5):
    	np.random.seed(1)
    	W1 = parameters["W1"]
    	b1 = parameters["b1"]
    	W2 = parameters["W2"]
    	b2 = parameters["b2"]
    	W3 = parameters["W3"]
    	b3 = parameters["b3"]
    	
    	Z1 = np.dot(W1, X) + b1
    	A1 = relu(Z1)
    	D1 = np.random.rand(A1.shape[0], A1.shape[1])
    	D1 = (D1 < keep_prob).astype(int)
    	A1 = A1 * D1
    	A1 /= keep_prob
    	
    	Z2 = np.dot(W2, A1) + b2
    	A2 = relu(Z2)
    	
    	D2 = np.random.rand(A2.shape[0], A2.shape[1])
    	D2 = (D2 < keep_prob).astype(int)
    	A2 = A2 * D2
    	A2 /= keep_prob
    	
    	Z3 = np.dot(W3,A2) +b3
    	A3 = sigmoid(Z3)
    	
    	cache = (Z1, D1, A1, W1, b1, Z2, D2, A2, W2, b2, Z3, A3, W3, b3)
    	
    	return A3, cache
    ```
    
- **Backwarding with Dropout**
    
    
    마지막 출력층에서는 dropout을 시행하지않고, 히든레이어 2개에서 Dropout을 시행했기때문에, 2개의 레이어에 대해서 활성함수 결과값이 0이상인 값들에 한해서만, 역전파를 수행해주면된다.
    
    ```python
    def backward_propagation_with_dropout(X, Y, cache, keep_prob):
    
    	m = X.shape[1]
    	(Z1, D1, A1, W1, b1, Z2, D2, A2, W2, b2, Z3, A3, W3, b3) = cache
    	
    	dZ3 = A3 - Y
    	dW3 = 1./m * np.dot(dZ3, A2.T)
    	db3 = 1./m * np.sum(dZ3, axis=1, keepdims=True)
    	dA2 = np.dot(W3.T, dZ3)
    	
    	dA2 = dA2 * D2
    	dA2 /= keep_prob
    	
    	
    	dZ2 = np.multiply(dA2, np.int64(A2 > 0))
    	dW2 = 1./m * np.dot(dZ2, A1.T)
    	db2 = 1./m * np.sum(dZ2, axis=1, keepdims=True)
    	
    	dA1 = np.dot(W2.T, dZ2)
    	dA1 = dA1 * D1
    	dA1 /= keep_prob
    	
    	dZ1 = np.multiply(dA1, np.int64(A1 > 0))
    	dW1 = 1./m * np.dot(dZ1, X.T)
    	db1 = 1./m * np.sum(dZ1, axis=1, keepdims=True)
    	    
    	gradients = {"dZ3": dZ3, "dW3": dW3, "db3": db3,"dA2": dA2,
    	             "dZ2": dZ2, "dW2": dW2, "db2": db2, "dA1": dA1, 
    	             "dZ1": dZ1, "dW1": dW1, "db1": db1}
    	    
    	return gradients
    ```
    
    ```python
    parameters = model(train_X, train_Y, keep_prob = 0.86, learning_rate = 0.3)
    
    print ("On the train set:")
    predictions_train = predict(train_X, train_Y, parameters)
    print ("On the test set:")
    predictions_test = predict(test_X, test_Y, parameters)
    ```
    
    ```
    Cost after iteration 0: 0.6543912405149825
    Cost after iteration 10000: 0.0610169865749056
    Cost after iteration 20000: 0.060582435798513114
    
    On the train set:
    Accuracy: 0.9289099526066351
    On the test set:
    Accuracy: 0.95
    ```
    
    ![Untitled 7](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/7dd6a1ac-1918-4f91-954e-2aad657f2e55)
    
    ![Untitled 8](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/5cf9a550-7212-4d0f-a9d8-cd9d16c1aa54)
    

> **결과**
> 

| 수행 | train ACC | test ACC |
| --- | --- | --- |
| non regularization | 95 | 91.5 |
| L2-Regularlization | 94 | 93 |
| NN with Dropout | 93 | 95 |

정규화는 과적합을 방지할 수 있으며, L2 정규화와 Dropout기법은 효과적인 정규화 기법이다.