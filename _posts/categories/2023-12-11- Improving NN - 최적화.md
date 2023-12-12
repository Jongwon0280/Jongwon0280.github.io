---
layout: single
title: Improving NN - 최적화
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-11
---

## Improving NN - 최적화

> **목적**
> 

다양한 최적화기법을 통해 수렴속도를 빠르게하고 오버슈팅을 회피한다.

> **최적화**
> 

- **경사하강법**
    
    
    `W = W - alpha * dW[L]`
    
    `B = B - alpha * dB[L]`
    
    ```python
    def update_parameters_with_gd(parameters, grads, learning_rate):
    	L = len(parameters) // 2 # 레이어개수를 세기위함. //2는 w,b를 포함하여 2개씩 레이어당 있기때문이다.
    	for l in range(1,L+1):
    		parameters["W" + str(l)]-=learning_rate * grads['dW'+str(l)]
    		parameters["b" + str(l)]-=learning_rate * grads['db'+str(l)]
    		
    	return parameters
    ```
    
    경사하강법은 모든 데이터 샘플을 한번에 처리하는 Batch방식과 한데이터샘플마다 Gradient을 구하는 SGD, 그 중간을 취하는 Mini-Batch Gradient Descenting이 있다.
    
    ![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/22f30a22-85da-47b3-8dbf-f90874036e16)
    
    ![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/9ae37b0d-57fe-4f7c-92d8-04b838c8100f)
    

- **Mini Batch Gradient Descenting**

1. **Shuffle**
    
    데이터샘플을 무작위로 섞는다. 라벨도 동일한 순서로 섞어야한다.
    
    ![Untitled 2](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/bcebf2a5-0208-4044-86b9-71bf89dfdd9d)
    
2. **Partition**
    
    지정한 배치사이즈 단위로 데이터샘플을 나눈다.
    
    ![Untitled 3](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/04c0b192-91ac-46f8-a403-a9c94d116c57)
    
3. **Final Batch**
    
    마지막 배치는 배치사이즈를 충족할 수 없을 수도 있기때문에, 전체 데이터샘플을 배치사이즈로 나눈 몫만큼의 데이터사이즈를 전체 데이터샘플개수에서 빼준 값을 마지막 batch로 지정한다.
    

```python
def random_mini_batches(X,Y, mini_batch_size = 64, seed = 0):

np.random.seed(seed)
m = X.shape[1]
mini_batches = []

permutation = list(np.random.permutation(m))
shuffled_X = X[:, permutation]
shuffled_Y = Y[:, permutation].reshape((1,m))

inc = mini_batch_size

num_complete_minibatches = math.floor(m / mini_batch_size) #최종미니배치개수

for k in range( 0, num_complete_minibatches):

	mini_batch_X= shuffled_X[:, k*mini_batch_size :  k*mini_batch_size + mini_batch_size]
  mini_batch_Y= shuffled_Y[:, k*mini_batch_size :  k*mini_batch_size + mini_batch_size].reshape((1, mini_batch_size))

	mini_batch = (mini_batch_X, mini_batch_Y)
  mini_batches.append(mini_batch)

if m % mini_batch_size != 0:
	mini_batch_X= shuffled_X[:,(num_complete_minibatches)*mini_batch_size : ]
  mini_batch_Y= shuffled_Y[:, (num_complete_minibatches)*mini_batch_size : ].reshape((1,mini_batch_X.shape[1] ))
	mini_batch = (mini_batch_X, mini_batch_Y)
  mini_batches.append(mini_batch)

return mini_batches
```

보통 미니배치 사이즈는 짝수로 지정한다. 16, 32, 64,128 …

- **Momentum**
    
    
    이전 그래디언트를 통해 지수가중평균을 통해 최저점에 더 빠르게 수렴하도록 할 수 있다.
    
     
    
    ![Untitled 4](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/6df95ff9-1e8a-4e94-95f3-24410f02a75a)
    
    - **velocity 초기화**
        
        각 뉴럴네트워크의 속도 가중치들을 초기화시켜준다.
        
        ```python
        def initialize_velocity(parameters):
        	L= len(parameters)//2
        	v={}
        	
        	for l in range(1,L+1):
        		v["dW" + str(l)] = np.zeros((parameters["dW"+str(l)].shape[0],parameters["dW"+str(l)].shape[1]))
        		v["db" + str(l)] = np.zeros((parameters["db"+str(l)].shape[0],1))
        	return v
        	
        ```
        
    - v**파라미터 수정**
        
        beta VdW는 속도역할을 담당하며, (1-beta)dW[l]은 가속기 역할이다.
        
        지수가중이동평균을 통해 전역최소점으로 이어지는 가로방향의 속도는 유지될것이며, 그에반해 세로축진동은 양수방향과 음수방향의 중심으로 잡혀 진동이 약해진다.
        
        ![Untitled 5](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/3e193816-770b-46f2-a163-f605fe1a46ca)
        
        ```python
        def update_parameters_with_momentum(parameters, grads, v, beta, learning_rate):
        
        	L = len(parameters) // 2
        	
        	for l in range(1, L + 1):
        		v["dW" + str(l)] = beta * v["dW" + str(l)] + ((1-beta) * grads['dW' + str(l)])
        	  v["db" + str(l)] = beta * v["db" + str(l)] + ((1-beta) * grads['db' + str(l)])
        		parameters["W" + str(l)] -= learning_rate * v["dW" + str(l)]
        	  parameters["b" + str(l)] -= learning_rate * v["db" + str(l)]
        	return parameters, v
        ```
        
        Beta는 보통 0.9로 사용하며, 튜닝하지 않는다. 
        
- **Adam**
    
    
    Adam은 Momentum과 RMSprops의 조합형태이다. 아래식에서 알수 있듯이, v,s를 통해 각각 계산하여, 최적화 시 분모에는 s 분자에는 v를 사용한다.
    
    ![Untitled 6](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/1d0177bb-215d-4835-9e18-f2bc424133f6)
    
    - **v, s parameters 초기화**
        
        v,s에 대해 0으로 초기화시켜준다.
        
        ```python
        def initialize_adam(parameters) :
        L = len(parameters) // 2 # number of layers in the neural networks
        v = {}
        s = {}
        
        for l in range(1, L + 1):
        
            v["dW" + str(l)] = np.zeros((parameters['W' + str(l)].shape[0],parameters['W' + str(l)].shape[1]))
            v["db" + str(l)] = np.zeros((parameters['b' + str(l)].shape[0],1))
            s["dW" + str(l)] = np.zeros((parameters['W' + str(l)].shape[0],parameters['W' + str(l)].shape[1]))
            s["db" + str(l)] = np.zeros((parameters['b' + str(l)].shape[0],1))
        
        return v, s
        ```
        
    - **paramters 수정**
        
        corrected의 역할은 초기 0으로 시작되는 bias에 대한 correction이다.
        
        ```python
        def update_parameters_with_adam(parameters, grads, v, s, t, learning_rate = 0.01,
                                  beta1 = 0.9, beta2 = 0.999,  epsilon = 1e-8):
        
        	L = len(parameters) // 2                 # number of layers in the neural networks
        	v_corrected = {}                         # Initializing first moment estimate, python dictionary
        	s_corrected = {}                         # Initializing second moment estimate, python dictionary
        	
        	# Perform Adam update on all parameters
        	for l in range(1, L + 1):
        	  # Moving average of the gradients. Inputs: "v, grads, beta1". Output: "v".
        	  # (approx. 2 lines)
        	  # v["dW" + str(l)] = ...
        	  # v["db" + str(l)] = ...
        	  # YOUR CODE STARTS HERE
        		  v["dW" + str(l)] = beta1 * v["dW" + str(l)] + ((1-beta1)* grads['dW' + str(l)])
        		  v["db" + str(l)] =beta1 * v["db" + str(l)] + ((1-beta1)* grads['db' + str(l)])
        	  # YOUR CODE ENDS HERE
        	
        	  # Compute bias-corrected first moment estimate. Inputs: "v, beta1, t". Output: "v_corrected".
        	  # (approx. 2 lines)
        	  # v_corrected["dW" + str(l)] = ...
        	  # v_corrected["db" + str(l)] = ...
        	  # YOUR CODE STARTS HERE
        		  v_corrected["dW" + str(l)] = v["dW" + str(l)] / (1-(beta1**t))
        		  v_corrected["db" + str(l)] = v["db" + str(l)] / (1-(beta1**t))
        	  
        	  # YOUR CODE ENDS HERE
        	
        	  # Moving average of the squared gradients. Inputs: "s, grads, beta2". Output: "s".
        	  #(approx. 2 lines)
        	  # s["dW" + str(l)] = ...
        	  # s["db" + str(l)] = ...
        	  # YOUR CODE STARTS HERE
        		  s["dW" + str(l)] = beta2 * s["dW" + str(l)] + ((1-beta2)* (np.square(grads['dW' + str(l)])))
        		  s["db" + str(l)] = beta2 * s["db" + str(l)] + ((1-beta2)* (np.square(grads['db' + str(l)])))
        		  
        	  # YOUR CODE ENDS HERE
        	
        	  # Compute bias-corrected second raw moment estimate. Inputs: "s, beta2, t". Output: "s_corrected".
        	  # (approx. 2 lines)
        	  # s_corrected["dW" + str(l)] = ...
        	  # s_corrected["db" + str(l)] = ...
        	  # YOUR CODE STARTS HERE
        		  s_corrected["dW" + str(l)] = s["dW" + str(l)] / (1-(beta2**t))
        		  s_corrected["db" + str(l)] = s["db" + str(l)] / (1-(beta2**t))
        	  
        	  # YOUR CODE ENDS HERE
        	
        	  # Update parameters. Inputs: "parameters, learning_rate, v_corrected, s_corrected, epsilon". Output: "parameters".
        	  # (approx. 2 lines)
        	  # parameters["W" + str(l)] = ...
        	  # parameters["b" + str(l)] = ...
        	  # YOUR CODE STARTS HERE
        		  parameters["W" + str(l)] -= (learning_rate*(v_corrected["dW" + str(l)])/(np.sqrt(s_corrected["dW" + str(l)])+epsilon))
        		  parameters["b" + str(l)] -= (learning_rate*(v_corrected["db" + str(l)])/(np.sqrt(s_corrected["db" + str(l)])+epsilon))
        	  
        	  # YOUR CODE ENDS HERE
        	
        	return parameters, v, s, v_corrected, s_corrected
        ```
        

- **평가**
    
    ```python
    def model(X, Y, layers_dims, optimizer, learning_rate = 0.0007, mini_batch_size = 64, beta = 0.9,
              beta1 = 0.9, beta2 = 0.999,  epsilon = 1e-8, num_epochs = 5000, print_cost = True):
        """
        3-layer neural network model which can be run in different optimizer modes.
        
        Arguments:
        X -- input data, of shape (2, number of examples)
        Y -- true "label" vector (1 for blue dot / 0 for red dot), of shape (1, number of examples)
        optimizer -- the optimizer to be passed, gradient descent, momentum or adam
        layers_dims -- python list, containing the size of each layer
        learning_rate -- the learning rate, scalar.
        mini_batch_size -- the size of a mini batch
        beta -- Momentum hyperparameter
        beta1 -- Exponential decay hyperparameter for the past gradients estimates 
        beta2 -- Exponential decay hyperparameter for the past squared gradients estimates 
        epsilon -- hyperparameter preventing division by zero in Adam updates
        num_epochs -- number of epochs
        print_cost -- True to print the cost every 1000 epochs
    
        Returns:
        parameters -- python dictionary containing your updated parameters 
        """
    
        L = len(layers_dims)             # number of layers in the neural networks
        costs = []                       # to keep track of the cost
        t = 0                            # initializing the counter required for Adam update
        seed = 10                        # For grading purposes, so that your "random" minibatches are the same as ours
        m = X.shape[1]                   # number of training examples
        
        # Initialize parameters
        parameters = initialize_parameters(layers_dims)
    
        # Initialize the optimizer
        if optimizer == "gd":
            pass # no initialization required for gradient descent
        elif optimizer == "momentum":
            v = initialize_velocity(parameters)
        elif optimizer == "adam":
            v, s = initialize_adam(parameters)
        
        # Optimization loop
        for i in range(num_epochs):
            
            # Define the random minibatches. We increment the seed to reshuffle differently the dataset after each epoch
            seed = seed + 1
            minibatches = random_mini_batches(X, Y, mini_batch_size, seed)
            cost_total = 0
            
            for minibatch in minibatches:
    
                # Select a minibatch
                (minibatch_X, minibatch_Y) = minibatch
    
                # Forward propagation
                a3, caches = forward_propagation(minibatch_X, parameters)
    
                # Compute cost and add to the cost total
                cost_total += compute_cost(a3, minibatch_Y)
    
                # Backward propagation
                grads = backward_propagation(minibatch_X, minibatch_Y, caches)
    
                # Update parameters
                if optimizer == "gd":
                    parameters = update_parameters_with_gd(parameters, grads, learning_rate)
                elif optimizer == "momentum":
                    parameters, v = update_parameters_with_momentum(parameters, grads, v, beta, learning_rate)
                elif optimizer == "adam":
                    t = t + 1 # Adam counter
                    parameters, v, s, _, _ = update_parameters_with_adam(parameters, grads, v, s,
                                                                   t, learning_rate, beta1, beta2,  epsilon)
            cost_avg = cost_total / m
            
            # Print the cost every 1000 epoch
            if print_cost and i % 1000 == 0:
                print ("Cost after epoch %i: %f" %(i, cost_avg))
            if print_cost and i % 100 == 0:
                costs.append(cost_avg)
                    
        # plot the cost
        plt.plot(costs)
        plt.ylabel('cost')
        plt.xlabel('epochs (per 100)')
        plt.title("Learning rate = " + str(learning_rate))
        plt.show()
    
        return parameters
    ```
    
    - **Mini-Batch GD**
        
        
        ![Untitled 7](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/568fb90f-a9c1-4995-ba8c-861365e9402a)
        
        ![Untitled 8](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/a50c092c-9f02-434c-a779-9bdc3fa74cf2)
        
    
    ```
    Cost after epoch 0: 0.702405
    Cost after epoch 1000: 0.668101
    Cost after epoch 2000: 0.635288
    Cost after epoch 3000: 0.600491
    Cost after epoch 4000: 0.573367
    
    Accuracy: 0.7166666666666667
    ```
    
    - **Mini-Batch with Momentum**
        
        
        ![Untitled 9](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/0be2638c-0319-4a30-91f6-cd8833d0b1e7)
        
        ![Untitled 10](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/6ac82aea-be58-438d-ad6b-c6baea0f8543)
        
        ```
        Cost after epoch 0: 0.702413
        Cost after epoch 1000: 0.668167
        Cost after epoch 2000: 0.635388
        Cost after epoch 3000: 0.600591
        Cost after epoch 4000: 0.573444
        
        Accuracy: 0.7166666666666667
        ```
        
    - **Mini-Batch with Adam**
        
        
        ![Untitled 11](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/af813183-14b3-42a7-b1e1-f74f5b284f45)
        
        ![Untitled 12](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/bfe70615-acfc-4d3e-89ad-face60a4a061)
        
        ```
        Cost after epoch 0: 0.702104
        Cost after epoch 1000: 0.644240
        Cost after epoch 2000: 0.594380
        Cost after epoch 3000: 0.560040
        Cost after epoch 4000: 0.532859
        
        Accuracy: 0.7566666666666667
        ```