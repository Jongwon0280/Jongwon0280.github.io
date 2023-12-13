---
layout: single
title:  NMT - with Attention
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-12
---

## NMT - with Attention

> **목적**
> 

**"25th of June, 2009"** 인간의 표시한 날짜형태를 기계가 읽을 수 있는 DateFormat으로 변환한다.  **"2009-06-25”**

LSTM과 attention메커니즘을 활용하여 문제를 해결한다.

> **데이터셋**
> 

데이터 셋은 10,000개의 인간의 표시한 날짜데이터와, 이를 DateFormat으로 바꾼 라벨데이터이다.

```
[('9 may 1998', '1998-05-09'),
 ('10.11.19', '2019-11-10'),
 ('9/10/70', '1970-09-10'),
 ('saturday april 28 1990', '1990-04-28'),
 ('thursday january 26 1995', '1995-01-26'),
 ('monday march 7 1983', '1983-03-07'),
 ('sunday may 22 1988', '1988-05-22'),
 ('08 jul 2008', '2008-07-08'),
 ('8 sep 1999', '1999-09-08'),
 ('thursday january 1 1981', '1981-01-01')]
```

> **코드**
> 

- **패키지**
    
    
    ```python
    from tensorflow.keras.layers import Bidirectional, Concatenate, Permute, Dot, Input, LSTM, Multiply
    from tensorflow.keras.layers import RepeatVector, Dense, Activation, Lambda
    from tensorflow.keras.optimizers import Adam
    from tensorflow.keras.utils import to_categorical
    from tensorflow.keras.models import load_model, Model
    import tensorflow.keras.backend as K
    import tensorflow as tf
    import numpy as np
    
    from faker import Faker
    import random
    from tqdm import tqdm
    from babel.dates import format_date
    from nmt_utils import *
    import matplotlib.pyplot as plt
    %matplotlib inline
    ```
    
- **데이터준비**
    
    
    인간이 표시한 날짜의 최대 시퀸스길이는 30이며, DateFormat의 최대 시퀸스길이는 10이므로, 데이터를 3차원으로 준비한다.
    
    ```python
    Tx = 30
    Ty = 10
    X, Y, Xoh, Yoh = preprocess_data(dataset, human_vocab, machine_vocab, Tx, Ty)
    
    print("X.shape:", X.shape)
    print("Y.shape:", Y.shape)
    print("Xoh.shape:", Xoh.shape)
    print("Yoh.shape:", Yoh.shape)
    print(machine_vocab)
    ```
    
    ```
    X.shape: (10000, 30)
    Y.shape: (10000, 10)
    Xoh.shape: (10000, 30, 37)
    Yoh.shape: (10000, 10, 11)
    {'-': 0, '0': 1, '1': 2, '2': 3, '3': 4, '4': 5, '5': 6, '6': 7, '7': 8, '8': 9, '9': 10}
    ```
    

```python
index = 0
print("Source date:", dataset[index][0])
print("Target date:", dataset[index][1])
print()
print("Source after preprocessing (indices):", X[index])
print("Target after preprocessing (indices):", Y[index])
print()
print("Source after preprocessing (one-hot):", Xoh[index])
print("Target after preprocessing (one-hot):", Yoh[index])
```

```
Source date: 9 may 1998
Target date: 1998-05-09

Source after preprocessing (indices): [12  0 24 13 34  0  4 12 12 11 36 36 36 36 36 36 36 36 36 36 36 36 36 36
 36 36 36 36 36 36]
Target after preprocessing (indices): [ 2 10 10  9  0  1  6  0  1 10]

Source after preprocessing (one-hot): [[0. 0. 0. ... 0. 0. 0.]
 [1. 0. 0. ... 0. 0. 0.]
 [0. 0. 0. ... 0. 0. 0.]
 ...
 [0. 0. 0. ... 0. 0. 1.]
 [0. 0. 0. ... 0. 0. 1.]
 [0. 0. 0. ... 0. 0. 1.]]
Target after preprocessing (one-hot): [[0. 0. 1. 0. 0. 0. 0. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 1.]
 [0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 1.]
 [0. 0. 0. 0. 0. 0. 0. 0. 0. 1. 0.]
 [1. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0.]
 [0. 1. 0. 0. 0. 0. 0. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0. 0. 1. 0. 0. 0. 0.]
 [1. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0.]
 [0. 1. 0. 0. 0. 0. 0. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 1.]]
```

- **Attention**
    
    인코더와 디코더의 사용할 모델은 양방향 LSTM모델이다. Attention메커니즘은 다음그림과 같이 작동한다.
    
    ![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/29f5bdd4-03f2-4fd3-be51-a3e852d734a1)
    
    인코더의 시퀸스별 히든스테이트 a<t>와 디코더의 출력시점이전의 히든스테이트 s<t-1>을 concat한후 선형레이어를 통해 attention 스코어를 얻는다. 출력 차원으로 (Tx, 1)의 값을 얻을 수 있고, 이를 softmax를 통해 확률분포로 변환한다. 
    
    ![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/e6088d7a-4ec2-419f-aa52-a3cd415b8b84)
    
    이를 인코더의 각시점별 히든스테이트 a<1…tx>와 가중합을 구하여 디코더의 입력으로 넣어준다.
    
    ```python
    #s<t-1>을 각 a<1...tx>에 concat하기 위함이다.
    repeator = RepeatVector(Tx)
    # concat한다.
    concatenator = Concatenate(axis=-1)
    # 선형레이어를 통과시킨다.
    densor1 = Dense(10, activation = "tanh")
    # output (Tx,)
    densor2 = Dense(1, activation = "relu")
    activator = Activation(softmax, name='attention_weights') # We are using a custom softmax(axis = 1) loaded in this notebook
    #  a<1...tx>와 attention스코어를 가중합하여, 
    # t시점의 디코더의 입력으로 넣는다.
    dotor = Dot(axes = 1)
    ```
    
    ```python
    def one_step_attention(a, s_prev):
        """
        Performs one step of attention: Outputs a context vector computed as a dot product of the attention weights
        "alphas" and the hidden states "a" of the Bi-LSTM.
        
        Arguments:
        a -- hidden state output of the Bi-LSTM, numpy-array of shape (m, Tx, 2*n_a)
        s_prev -- previous hidden state of the (post-attention) LSTM, numpy-array of shape (m, n_s)
        
        Returns:
        context -- context vector, input of the next (post-attention) LSTM cell
        """
        
        
        s_prev = repeator(s_prev)
    
        concat = concatenator([a,s_prev])
        
        e = densor1(concat)
        
        energies = densor2(e)
        
        alphas = activator(energies)
        
        context = dotor([alphas,a])
        
        
        return context
    ```
    
    ```python
    #인코더의 히든스테이트 유닛과 디코더의 히든스테이트 유닛 정의
    n_a = 32 
    n_s = 64
    # 디코더에 사용할 LSTM 정의
    post_activation_LSTM_cell = LSTM(n_s, return_state = True) 
    output_layer = Dense(len(machine_vocab), activation=softmax)
    ```
    
    ```python
    def modelf(Tx, Ty, n_a, n_s, human_vocab_size, machine_vocab_size):
        """
        Arguments:
        Tx -- length of the input sequence
        Ty -- length of the output sequence
        n_a -- hidden state size of the Bi-LSTM
        n_s -- hidden state size of the post-attention LSTM
        human_vocab_size -- size of the python dictionary "human_vocab"
        machine_vocab_size -- size of the python dictionary "machine_vocab"
    
        Returns:
        model -- Keras model instance
        """
        
        
        
        X = Input(shape=(Tx, human_vocab_size))
        
        s0 = Input(shape=(n_s,), name='s0')
        
        c0 = Input(shape=(n_s,), name='c0')
        
        s = s0
        
        c = c0
        
        outputs = []
        
        
        # Step 1: 인코더정의 
        a = Bidirectional(LSTM(units=n_a ,return_sequences=True))(X)
        
        # Step 2: Seq2Seq에 맞게 루프 정의
        for t in range(Ty):
        
            # Step 2.A: 순차적으로 디코더의 input 계산
            context = one_step_attention(a,s)
            
            # Step 2.B: LSTM의 이전 a<t-1> , c<t-1> , 계산한 attention입력을 넣어줌
            _, s, c = post_activation_LSTM_cell(inputs=context ,initial_state=[s, c])
            
            # Step 2.C: 선형레이어를 통해 softmax거쳐서, 한자리씩 출력하기위함.
            out = output_layer(s)
            
            # Step 2.D: 결과물 출력
            outputs.append(out)
        
        # Step 3: 모델정의, 입력 / 출력형식
        model = Model(inputs=[X,s0,c0],outputs=outputs)
        
        
        
        return model
    ```
    
    | Total params | 52,960 |
    | --- | --- |
    | Trainable params | 52,960 |
    | Non-trainable params | 0 |
    | bidirectional_1's output shape | (None, 30, 64) |
    | repeat_vector_1's output shape | (None, 30, 64) |
    | concatenate_1's output shape | (None, 30, 128) |
    | attention_weights's output shape  | (None, 30, 1) |
    | dot_1's output shape | (None, 1, 64) |
    | dense_3's output shape | (None, 11) |
    
- **Compile Model**
    
    정의한 모델의 최적화 방식과 하이퍼파라미터를 지정해준다.
    
    ```python
    opt = Adam(lr=0.005,beta_1=0.9,beta_2=0.999,decay=0.01) # Adam(...) 
    model.compile(loss = 'categorical_crossentropy', optimizer =opt, metrics =['accuracy'])
    ```
    
    ```python
    s0 = np.zeros((m, n_s))
    c0 = np.zeros((m, n_s))
    outputs = list(Yoh.swapaxes(0,1))
    
    model.fit([Xoh, s0, c0], outputs, epochs=1, batch_size=100)
    ```
    
- **Result**
    
    
    ```python
    EXAMPLES = ['3 May 1979', '5 April 09', '21th of August 2016', 'Tue 10 Jul 2007', 'Saturday May 9 2018', 'March 3 2001', 'March 3rd 2001', '1 March 2001']
    s00 = np.zeros((1, n_s))
    c00 = np.zeros((1, n_s))
    for example in EXAMPLES:
        source = string_to_int(example, Tx, human_vocab)
        #print(source)
        source = np.array(list(map(lambda x: to_categorical(x, num_classes=len(human_vocab)), source))).swapaxes(0,1)
        source = np.swapaxes(source, 0, 1)
        source = np.expand_dims(source, axis=0)
        prediction = model.predict([source, s00, c00])
        prediction = np.argmax(prediction, axis = -1)
        output = [inv_machine_vocab[int(i)] for i in prediction]
        print("source:", example)
        print("output:", ''.join(output),"\n")
    ```
    
    ```
    source: 3 May 1979
    output: 1979-05-33
    
    source: 5 April 09
    output: 2009-04-05
    
    source: 21th of August 2016
    output: 2016-08-20
    
    source: Tue 10 Jul 2007
    output: 2007-07-10
    
    source: Saturday May 9 2018
    output: 2018-05-09
    
    source: March 3 2001
    output: 2001-03-03
    
    source: March 3rd 2001
    output: 2001-03-03
    
    source: 1 March 2001
    output: 2001-03-01
    ```