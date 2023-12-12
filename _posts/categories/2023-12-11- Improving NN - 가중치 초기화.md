---
layout: single
title: Improving NN - 가중치 초기화
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-11
---

## Improving NN - 가중치 초기화

> **목적**
> 

NN의 가중치 초기화를 통한 성능향상을 시도한다.

 

> **데이터셋**
> 

![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/3ec6d229-f477-4a9a-b215-f299be33f1f0)

도넛형으로 구성된 이진 분류 데이터셋이다.

> **Model**
> 

모델은 전 과정에서 구축한 3-layer NN을 사용한다.

```python
def model(X, Y, learning_rate = 0.01, num_iterations = 15000, print_cost = True, initialization = "he"):
    """
    Implements a three-layer neural network: LINEAR->RELU->LINEAR->RELU->LINEAR->SIGMOID.
    
    Arguments:
    X -- input data, of shape (2, number of examples)
    Y -- true "label" vector (containing 0 for red dots; 1 for blue dots), of shape (1, number of examples)
    learning_rate -- learning rate for gradient descent 
    num_iterations -- number of iterations to run gradient descent
    print_cost -- if True, print the cost every 1000 iterations
    initialization -- flag to choose which initialization to use ("zeros","random" or "he")
    
    Returns:
    parameters -- parameters learnt by the model
    """
        
    grads = {}
    costs = [] # to keep track of the loss
    m = X.shape[1] # number of examples
    layers_dims = [X.shape[0], 10, 5, 1]
    
    # Initialize parameters dictionary.
    if initialization == "zeros":
        parameters = initialize_parameters_zeros(layers_dims)
    elif initialization == "random":
        parameters = initialize_parameters_random(layers_dims)
    elif initialization == "he":
        parameters = initialize_parameters_he(layers_dims)

    # Loop (gradient descent)

    for i in range(num_iterations):

        # Forward propagation: LINEAR -> RELU -> LINEAR -> RELU -> LINEAR -> SIGMOID.
        a3, cache = forward_propagation(X, parameters)
        
        # Loss
        cost = compute_loss(a3, Y)

        # Backward propagation.
        grads = backward_propagation(X, Y, cache)
        
        # Update parameters.
        parameters = update_parameters(parameters, grads, learning_rate)
        
        # Print the loss every 1000 iterations
        if print_cost and i % 1000 == 0:
            print("Cost after iteration {}: {}".format(i, cost))
            costs.append(cost)
            
    # plot the loss
    plt.plot(costs)
    plt.ylabel('cost')
    plt.xlabel('iterations (per hundreds)')
    plt.title("Learning rate =" + str(learning_rate))
    plt.show()
    
    return parameters

```

> **weight** **초기화 방식**
> 

W[1] … W[L-1], W[L] / b[1] … b[L-1],b[L]의 가중치 초기화에 대한 방법이다.

- **Zero 초기화**
    
    
    np.zeros()를 활용하여 각 NN에 shape에 맞게 0으로 초기화한다.
    
    ```python
    def initialize_parameters_zeros(layers_dims):
    	parameters = {}
    	L = len(layers_dims)
    	for i in range(1,L):
    		parameters['W' + str(i)] = np.zeros((layers_dims[i],layers_dims[i-1]))
    		parameters['b' + str(i)] = np.zeros((layers_dims[i],1))
    	return parameters
    ```
    
    ```python
    parameters = model(train_X, train_Y, initialization = "zeros")
    print ("On the train set:")
    predictions_train = predict(train_X, train_Y, parameters)
    print ("On the test set:")
    predictions_test = predict(test_X, test_Y, parameters)
    ```
    
    ```
    Cost after iteration 0: 0.6931471805599453
    Cost after iteration 1000: 0.6931471805599453
    Cost after iteration 2000: 0.6931471805599453
    Cost after iteration 3000: 0.6931471805599453
    Cost after iteration 4000: 0.6931471805599453
    Cost after iteration 5000: 0.6931471805599453
    Cost after iteration 6000: 0.6931471805599453
    Cost after iteration 7000: 0.6931471805599453
    Cost after iteration 8000: 0.6931471805599453
    Cost after iteration 9000: 0.6931471805599453
    Cost after iteration 10000: 0.6931471805599455
    Cost after iteration 11000: 0.6931471805599453
    Cost after iteration 12000: 0.6931471805599453
    Cost after iteration 13000: 0.6931471805599453
    Cost after iteration 14000: 0.6931471805599453
    
    On the train set:
    Accuracy: 0.5
    On the test set:
    Accuracy: 0.5
    ```
    
    ![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/ed05bf0f-ba6c-403f-b5b6-60d956445cdb)
    
    **𝑎=𝑅𝑒𝐿𝑈(𝑧)=𝑚𝑎𝑥(0,𝑧)=0**
    
    초기 가중치가 0으로 초기화 되었으므로, a=0이며, 말단에 sigmoid를 적용하면 
    
    1/ 1+e^0 = 1/2 = ypred가 된다.
    
    이를 손실함수에 적용하면
    
    L(𝑎,𝑦)=−𝑦 ln (𝑦𝑝𝑟𝑒𝑑)−(1−𝑦) ln (1−𝑦𝑝𝑟𝑒𝑑)
    
    if y가 1이라면
    
    = - 1 ln ( 1/2) - (0) ln ( 1 - 0.5)
    
    = -ln(1/2) = 0.693
    
    if y가 0이라면
    
    = -ln(1/2) = 0.693
    
    예측이 0.5일 수 밖에 없기때문에, 동일한 손실을 출력할 수 밖에 없고, 이는 가중치를 변경하지 않게한다.
    
    **→ 절대 0으로 초기화해서는안된다.**
    
- **랜덤 초기화**
    
    
    np.random.randn(shape)을 사용하여 랜덤으로 w를 초기화하고, b는 zeros를 통해 초기화한다.
    
    ```python
    # np.random.randn은 정규분포
    # np.random.rand는 균일분포에서 랜덤으로 한다.
    
    def initialize_parameters_random(layers_dims):
    
    	np.random.seed(3)
    	parameters = {}
    	L = len(layers_dims)
    	for l in range(1,L):
    		parameters['W'+str(l)] = np.random.randn(layers_dims[l],layers_dims[l-1]) * 10
    		parameters['b'+str(l)] = np.zeros((layers_dims[l],1))
    	return parameters
    ```
    
    ```python
    parameters = model(train_X, train_Y, initialization = "random")
    print ("On the train set:")
    predictions_train = predict(train_X, train_Y, parameters)
    print ("On the test set:")
    predictions_test = predict(test_X, test_Y, parameters)
    ```
    
    ```
    Cost after iteration 0: inf
    Cost after iteration 1000: 0.6247924745506072
    Cost after iteration 2000: 0.5980258056061102
    Cost after iteration 3000: 0.5637539062842213
    Cost after iteration 4000: 0.5501256393526495
    Cost after iteration 5000: 0.5443826306793814
    Cost after iteration 6000: 0.5373895855049121
    Cost after iteration 7000: 0.47157999220550006
    Cost after iteration 8000: 0.39770475516243037
    Cost after iteration 9000: 0.3934560146692851
    Cost after iteration 10000: 0.3920227137490125
    Cost after iteration 11000: 0.38913700035966736
    Cost after iteration 12000: 0.3861358766546214
    Cost after iteration 13000: 0.38497629552893475
    Cost after iteration 14000: 0.38276694641706693
    
    On the train set:
    Accuracy: 0.83
    On the test set:
    Accuracy: 0.86
    ```
    
    초기 비용함수의 손실값이 inf로 나타나는 것은 가중치 초기화를 매우큰값으로 시작했기에, 손실함수가 무한대에 비슷한 손실을 가져 표현을 못하게 된것이다. 즉, 높은 랜덤 가중치 값은 최적화의 속도를 느리게 하기에 작은 random값을 곱해주는것이 중요하다.
    
    ![Untitled 2](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/0fd60038-bab5-4d20-a6de-a0293a089747)
    
    ![Untitled 3](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/e891c881-a910-4def-b125-18297eeced8d)
    

- **He 초기화**
    
    Xavier 초기화라고도 불리는 초기화방식으로 정규분포로 부터 뽑은 랜덤value에
    
    sqrt(2./layers_dims[l-1])를 곱해서 사용한다.
    
    ```python
    def initialize_parameters_he(layers_dims):
    		np.random.seed(3)
        parameters = {}
        L = len(layers_dims) - 1 
         
        for l in range(1, L + 1):
           
            parameters['W' + str(l)] = np.random.randn(layers_dims[l],layers_dims[l-1]) *np.sqrt(2./layers_dims[l-1])
            parameters['b' + str(l)] = np.zeros((layers_dims[l],1))
            
            
        return parameters
    
    ```
    
    ```python
    parameters = model(train_X, train_Y, initialization = "he")
    print ("On the train set:")
    predictions_train = predict(train_X, train_Y, parameters)
    print ("On the test set:")
    predictions_test = predict(test_X, test_Y, parameters)
    ```
    
    ```
    Cost after iteration 0: 0.8830537463419761
    Cost after iteration 1000: 0.6879825919728063
    Cost after iteration 2000: 0.6751286264523371
    Cost after iteration 3000: 0.6526117768893805
    Cost after iteration 4000: 0.6082958970572938
    Cost after iteration 5000: 0.5304944491717495
    Cost after iteration 6000: 0.4138645817071794
    Cost after iteration 7000: 0.3117803464844441
    Cost after iteration 8000: 0.23696215330322562
    Cost after iteration 9000: 0.1859728720920684
    Cost after iteration 10000: 0.15015556280371808
    Cost after iteration 11000: 0.12325079292273551
    Cost after iteration 12000: 0.09917746546525937
    Cost after iteration 13000: 0.08457055954024283
    Cost after iteration 14000: 0.07357895962677366
    
    On the train set:
    Accuracy: 0.9933333333333333
    On the test set:
    Accuracy: 0.96
    ```
    
    ![Untitled 4](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/01e36af9-dd71-4e20-9f53-7193bf92767f)
    
    ![Untitled 5](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/93708082-b092-46ec-817e-869e213c9023)

    

> **결론**
> 

모델은 공통적으로 3 layer- NN

활성화 Relu, 출력층 sigmoid , 손실함수 binary-crossentropy를 사용.

| 초기화방식 | 훈련 정확도 | 평가 |
| --- | --- | --- |
| zero init | 50% | 동일한 손실함수 값으로 가중치 수정 불가능 |
| large random init | 83% | 너무 큰 weight로 최적화속도문제 |
| He init | 99% | 가장 효율적이다. |