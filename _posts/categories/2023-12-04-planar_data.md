---
layout: single
title: Classification - Planar Data
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-04
---

## NN을 통한 Planar 데이터셋 이진분류


> **목적**
> 

주어진 데이터를 이진분류하며, 은닉유닛을 유동적으로 조정해가며 결과를 확인한다.

> **데이터셋**
> 

![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/03fdd91a-cf0b-4bc5-aac4-5a10994ad91f)

| X | 2,400 |
| --- | --- |
| Y | 1,400 |
| m | 400 |

> **Simple Logistic Regression**
> 

sklearn의 logistic regression모델을 불러와 결과를 확인해보자.

```python
clf = sklearn.linear_model.LogisticRegressionCV()
clf.fit(X.T, Y.T)

plot_decision_boundary(lambda x: clf.predict(x), X, Y)
plt.title("Logistic Regression")

# Print accuracy
LR_predictions = clf.predict(X.T)
print ('Accuracy of logistic regression: %d ' % float((np.dot(Y,LR_predictions) + np.dot(1-Y,1-LR_predictions))/float(Y.size)*100) +
       '% ' + "(percentage of correctly labelled datapoints)")

```

`Accuracy of logistic regression: 47 % (percentage of correctly labelled datapoints)`

![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/3e50b701-2768-453b-9f37-bc0e713601ec)

정확도가 47%로 상당히 낮은 수치이다. 이유는 LR은 비선형 데이터셋에서 잘 작동하지 않기 때문이다. 이를 NN으로 개선해보자.

> **NN Model**
> 

![Untitled 2](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/fdb88492-dce7-493d-ac79-7e10a0390bfc)

![Untitled 3](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/d11ecf30-13a2-4f96-8d45-381d1a996fe4)

활성화 함수는 tanh를 사용하고, 출력층에서는 sigmoid, 손실함수는 2진크로스엔트로피를 사용한다.

- **레이어 크기**
    
    ```python
    def layer_sizes(X,Y):
    	n_x = X.shape[0]
    	n_h = 4
    	n_y = Y.shape[0]
    	    
    	    
    	return (n_x, n_h, n_y)
    	
    ```
    
- **모델 파라미터 초기화**
    
    ```python
    def initialize_parameters(n_x, n_h, n_y):
    	W1 = np.random.randn(n_h,n_x)*0.01 # (4,5)
      b1 = np.zeros((n_h,1)) # ( 4,1)
      W2 = np.random.randn(n_y,n_h)*0.01 # (2, 4)
      b2 = np.zeros((n_y,1)) # (2,1)
      
      
    
      parameters = {"W1": W1,
                    "b1": b1,
                    "W2": W2,
                    "b2": b2}
      
      return parameters
    ```
    

- **전향파(Forward)**
    
    ```python
    def forward_propagation(X,parameters):
    	W1 = parameters['W1']
      b1 = parameters['b1']
      W2 = parameters['W2']
      b2 = parameters['b2']
    
    	Z1 = np.dot(W1, X) + b1
      A1 = np.tanh(Z1)
      Z2 = np.dot(W2, A1) + b2
      A2 = sigmoid(Z2)
    
    	cache = {"Z1": Z1,
                 "A1": A1,
                 "Z2": Z2,
                 "A2": A2}
        
      return A2, cache
    ```
    
- **손실함수 , 비용함수**
    
    ![Untitled 4](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/ca43f8d1-e57f-4d5f-a5bc-fb9f5cbe41c8)
    
    ```python
    def compute_cost(A2, Y):
    	m = Y.shape[1]
    	logprobs = np.multiply(np.log(A2),Y) + np.multiply((1-Y),(np.log(1-A2)))
    	cost = - (1./m) * np.sum(logprobs)
    	cost = float(np.squeeze(cost))
    	return cost
    ```
    

- **후향파(Backpropagation)**
    
    ![Untitled 5](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/7a10bc78-d4f5-4e40-8792-fc4df12fd933)
    
    ```python
    
    def backward_propagation(parameters, cache, X, Y):
       
        m = X.shape[1]
        
        
        W1 = parameters['W1']
        W2 = parameters['W2']
        
        A1 = cache['A1']
        A2 = cache['A2']
        
        dZ2 = A2 - Y
        dW2 = (1./m)*np.dot(dZ2, A1.T)
        db2 = (1./m)*np.sum(dZ2,axis=1,keepdims=True)
        dZ1 = np.dot(W2.T, dZ2) * (1 - np.power(A1, 2))
        dW1 = (1./m)*np.dot(dZ1, X.T)
        db1 = (1./m)*np.sum(dZ1,axis=1,keepdims=True)
        
        
        grads = {"dW1": dW1,
                 "db1": db1,
                 "dW2": dW2,
                 "db2": db2}
        
        return grads
    ```
    
- **Update Parameters**
    
    ```python
    
    def update_parameters(parameters, grads, learning_rate = 1.2):
        
        W1 = parameters['W1']
        b1 = parameters['b1']
        W2 = parameters['W2']
        b2 = parameters['b2']
        
        
        dW1=grads['dW1']
        db1 = grads['db1']
        dW2=grads['dW2']
        db2 = grads['db2']
        
       
        W1 = W1 - learning_rate * dW1
        b1 = b1 - learning_rate * db1
        W2 = W2 - learning_rate * dW2
        b2 = b2 - learning_rate * db2
        
        parameters = {"W1": W1,
                      "b1": b1,
                      "W2": W2,
                      "b2": b2}
        
        return parameters
    ```
    

> **Model 통합**
> 

```python
def nn_model(X, Y, n_h, num_iterations = 10000, print_cost=False):
    
    
    np.random.seed(3)
    n_x = layer_sizes(X, Y)[0]
    n_y = layer_sizes(X, Y)[2]
    
    
    parameters = initialize_parameters(n_x,n_h,n_y)
    

    for i in range(0, num_iterations):
         
        
        A2 , cache = forward_propagation(X,parameters)
        cost = compute_cost(A2,Y)
        grads = backward_propagation(parameters, cache, X,Y)
        parameters = update_parameters(parameters, grads)
        
       
        if print_cost and i % 1000 == 0:
            print ("Cost after iteration %i: %f" %(i, cost))

    return parameters
```

- **예측하기**
    
    ```python
    
    def predict(parameters, X):
    
        A2, cache = forward_propagation(X,parameters)
        predictions =(A2 > 0.5)
        
        return predictions
    ```
    
- **결과**
    
    ```python
    parameters = nn_model(X, Y, n_h = 4, num_iterations = 10000, print_cost=True)
    plot_decision_boundary(lambda x: predict(parameters, x.T), X, Y)
    plt.title("Decision Boundary for hidden layer size " + str(4))
    ```
    
    ```
    Cost after iteration 0: 0.693162
    Cost after iteration 1000: 0.258625
    Cost after iteration 2000: 0.239334
    Cost after iteration 3000: 0.230802
    Cost after iteration 4000: 0.225528
    Cost after iteration 5000: 0.221845
    Cost after iteration 6000: 0.219094
    Cost after iteration 7000: 0.220661
    Cost after iteration 8000: 0.219409
    Cost after iteration 9000: 0.218485
    ```
    
    ![Untitled 6](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/e6d224b2-af77-4945-b2df-c3ae0d3f9584)
    
    | Accuracy | 90% |
    | --- | --- |
    
    히든유닛의 개수를 조정했을때 결과는 다음과 같다.
    
    ![Untitled 7](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/387943b2-dba6-4e40-a65e-f4f08f293cbc)
    
    ```
    Accuracy for 1 hidden units: 67.5 %
    Accuracy for 2 hidden units: 67.25 %
    Accuracy for 3 hidden units: 90.75 %
    Accuracy for 4 hidden units: 90.5 %
    Accuracy for 5 hidden units: 91.25 %
    ```