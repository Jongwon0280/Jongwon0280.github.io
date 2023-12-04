---
layout: single
title: LogReg - Cat Recognition
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-04
---

##로지스틱 회귀를 통한 고양이의 이진 분류 모델을 구축

> **목적**
> 

로지스틱 회귀를 통한 고양이의 이진 분류 모델을 구축

> **활용**
> 
- numpy
- logistic Regression ( Hand crafted)

> **데이터셋**
> 

209장의 (64,64,3)의 이미지이며, 고양이와 고양이가 아닌 이미지로 구성되어있어 0과 1의 라벨을 가지고 있다.

![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/c4844e90-6562-47f8-9d41-b7b19ecef40c)

| m_train | 209 |
| --- | --- |
| m_test | 50 |
| num_px | 64 |

> **Image2Vector**
> 

CNN이 아닌 Logistic Regression을 활용하기 위해서 입력을 벡터로 reshape이 필요하다.

```python
train_set_x_flatten = train_set_x_orig.reshape(train_set_x_orig.shape[0],-1).T
test_set_x_flatten = test_set_x_orig.reshape(test_set_x_orig.shape[0],-1).T

print ("train_set_x_flatten shape: " + str(train_set_x_flatten.shape))
print ("train_set_y shape: " + str(train_set_y.shape))
print ("test_set_x_flatten shape: " + str(test_set_x_flatten.shape))
print ("test_set_y shape: " + str(test_set_y.shape))

#train_set_x_flatten shape: (12288, 209)
#train_set_y shape: (1, 209)
#test_set_x_flatten shape: (12288, 50)
#test_set_y shape: (1, 50)
```

> **일반화**
> 

0-1 사이로 위치시켜주기위해 255로 나누어준다.

```python
train_set_x = train_set_x_flatten / 255.
test_set_x = test_set_x_flatten / 255.
```

> **모델 아키텍처**
> 

![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/47583f50-b3db-403e-9a9c-1b932543a9a8)
![Untitled 2](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/6c5a6a8a-0214-47cc-ab00-b64215cabb47)

> **모델 구현**
> 

- **sigmoid**
    
    ```python
    def sigmoid(z):
    	s= 1/ (1+np.exp(-z))
    	return s
    
    ```
    
- **가중치와 편향을 0으로 생성**
    
    ```python
    def initialize_with_zeros(dim):
    	w=np.zeros((dim,1),dtype='float32')
    	b=0.
    	return w,b
    
    ```
    
- **순전파 , 역전파**
    
    손실함수는 binary cross entropy를 사용하였고, 그에대한 미분값은 A-Y로 도출된다.
    
    ![Untitled 3](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/364c5e6b-6f97-4bdd-a036-7b08901a4503)
    
    ![Untitled 4](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/84449717-fa03-4926-ab4d-23ec7ddd944f)
    
    ```python
    def propagate(w,b,X,Y):
    	m=X.shape[1] #sample의 개수
    	A = sigmoid(np.dot(w.T,X) + b)
    	cost = -(1/m)*(np.sum(Y*np.log(A)+((1.-Y)*np.log(1.-A))))
    
    	dw = (1/m) * np.dot(X,(A-Y).T)
    	db = (1/m) * np.sum(A-Y)
    	cost = np.squeeze(np.array(cost))
    	
    	grads = {"dw": dw,
                 "db": db}
        
      return grads, cost
    	
    ```
    
- **최적화**
    
    w(new) = w(prev) - alpha * dw(prev)
    
    alpha는 학습률이다.
    
    ```python
    def optimize(w,b,X,Y,num_iterations = 100, learning_rate = 0.009, print_cost=False):
    	w = copy.deepcopy(w)
      b = copy.deepcopy(b)
        
      costs = []
    	for i in range(num_iterations):
    	        
    	        grads, cost = propagate(w, b, X, Y)
    	        
    	       
    	        dw = grads["dw"]
    	        db = grads["db"]
    	        
    	        
    	        w= w - (learning_rate * dw)
    	        b = b - (learning_rate * db)
    	        
    	        
    	        if i % 100 == 0:
    	            costs.append(cost)
    	        
    	         
    	            if print_cost:
    	                print ("Cost after iteration %i: %f" %(i, cost))
    	    
    	    params = {"w": w,
    	              "b": b}
    	    
    	    grads = {"dw": dw,
    	             "db": db}
    	    
    	    return params, grads, costs
    ```
    
- **예측**
    
    0.5 초과시 고양이, 0.5 이하일경우 고양이가 아니다.
    
    ```python
    def predict(w,b,X):
    	m = X.shape[1]
      Y_prediction = np.zeros((1, m))
      w = w.reshape(X.shape[0], 1)
    
    	A = sigmoid(np.dot(w.T,X) + b)
    	
    	for i in range(A.shape[1]):
    	       
    	            Y_prediction[0,i]=1
    	        else :
    	            Y_prediction[0,i]=0
    	        
    	        
    	    return Y_prediction
    ```
    

> **최종 모델**
> 

지금까지 구현한 모듈들을 하나로 통합한다.

```python

def model(X_train, Y_train, X_test, Y_test, num_iterations=2000, learning_rate=0.5, print_cost=False):
    
    w,b=initialize_with_zeros(X_train.shape[0])
    params, grads, costs = optimize(w, b, X_train, Y_train, num_iterations=num_iterations, learning_rate=learning_rate, print_cost=print_cost)
    w=params['w']
    b=params['b']
    
    Y_prediction_test=predict(w,b,X_test)
    Y_prediction_train=predict(w,b,X_train)
   
    if print_cost:
        print("train accuracy: {} %".format(100 - np.mean(np.abs(Y_prediction_train - Y_train)) * 100))
        print("test accuracy: {} %".format(100 - np.mean(np.abs(Y_prediction_test - Y_test)) * 100))

    
    d = {"costs": costs,
         "Y_prediction_test": Y_prediction_test, 
         "Y_prediction_train" : Y_prediction_train, 
         "w" : w, 
         "b" : b,
         "learning_rate" : learning_rate,
         "num_iterations": num_iterations}
    
    return d

logistic_regression_model = model(train_set_x, train_set_y, test_set_x, test_set_y, num_iterations=2000, learning_rate=0.005, print_cost=True)

#train accuracy: 99.04306220095694 %
#test accuracy: 70.0 %
```

`y = 1, you predicted that it is a "cat" picture.`

![Untitled 5](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/5b2218bc-f13c-4c5b-b8f0-93898aa92ebb)
![Untitled 6](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/4766bbd6-b469-40cc-8ecc-ccb4742e2852)
