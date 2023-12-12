---
layout: single
title:  RNN - Jazz Solo with LSTM
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-11
---

## RNN - Jazz Solo with LSTM

> **목적**
> 

LSTM모델을 활용한 재즈음악생성

> **활용**
> 
- LSTM을 통한 음악 패턴 학습
- 학습된 LSTM모델을 통해 음악 생성

> **데이터셋**
> 

음악의 값이란 음높이와 지속시간을 포함한다.

여기서는 데이터셋에서 값을 얻고, RNN을 통한 값의 연속을 생성한다. 

90개의 고유한 값을 사용함.

```python
X, Y, n_values, indices_values, chords = load_music_utils('data/original_metheny.mid')
print('number of training examples:', X.shape[0])
print('Tx (length of sequence):', X.shape[1])
print('total # of unique values:', n_values)
print('shape of X:', X.shape)
print('Shape of Y:', Y.shape)
print('Number of chords', len(chords))
```

X,Y는 다음의 코드에 의해서 30의 길이를 가지는 60개의 서브샘플로 구성이됩니다.
corpus의 길이는 90이며, 각 샘플은 30의 길이를 가져야 하므로 **90-30=60**에서 랜덤으로 숫자를 뽑고, 30의 길이를 가지는 corp_data를 통해 X,Y를 생성합니다. y<t> = x<t+1>이므로, j==1일때부터, y<t-1> = x<t>를 통해 y<0> = x<1> … y<t-1> = x<Tx> 까지 진행하며 90개의 원핫벡터에 미리 찾은 idx위치에 1을 삽입합니다.

**결론적으로 각 시퀸스의 입력  x<t>의 타겟값은 x<t+1>이며 X<0> = Y<0>(X<1>)로써 y^<t>에 대해서 90개의 categorical cross-entropy를 통해 손실을 계산할 수 있습니다.**

```python
def data_processing(corpus, values_indices, m = 60, Tx = 30):
    # cut the corpus into semi-redundant sequences of Tx values
    Tx = Tx 
    N_values = len(set(corpus))
    np.random.seed(0)
    X = np.zeros((m, Tx, N_values), dtype=np.bool)
    Y = np.zeros((m, Tx, N_values), dtype=np.bool)
    for i in range(m):
#         for t in range(1, Tx):
        random_idx = np.random.choice(len(corpus) - Tx)
        corp_data = corpus[random_idx:(random_idx + Tx)]
        for j in range(Tx):
            idx = values_indices[corp_data[j]]
            if j != 0:
                X[i, j, idx] = 1
                Y[i, j-1, idx] = 1
    
    Y = np.swapaxes(Y,0,1)
    Y = Y.tolist()
    return np.asarray(X), np.asarray(Y), N_values
```

X : **(m, Tx, 90) Tx는 30초이며, 90개의 가능한 값들이 존재한다.**

Y : **(Ty, m, 90) X와 유사하지만, 순서만 변형하였다.**

> **모델**
> 

![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/2111de3b-981c-4313-90da-08b72f6b3a24)

- **3개의 레이어**
    
    
    ```python
    n_values = 90 # number of music values
    reshaper = Reshape((1, n_values))                 
    LSTM_cell = LSTM(n_a, return_state = True)  
    densor = Dense(n_values, activation='softmax')
    ```
    

> **모델 구축**
> 

```python
n_a = 64
# LSTM의 hidden state를 64로정의
```

- **입력 및 레이블**
    
    X : (m,Tx,90)
    
    Y : (Ty,m,90)
    
- **LSTM 설정**
    
    na = 64 차원
    
    a,c 모두 전체 차원은 ( na, m )이지만, m은 생각하지 않고 개별 샘플로 구현하여야한다.
    
    → (na , 1)
    
- **시퀸스 생성**
    
    이전시간 단계에서의 예측을 현재 시간단계의 입력으로 사용함.
    
    **x<t> = y<t-1>**
    

- **공유가중치**
    
    LSTM계층을 Tx번 호출한다. 모든 Tx단계에서 동일한 가중치를 공유한다.
    
    전역변수로 계층객체를 정의함으로써, 입력을 전파할때 호출함.
    
1. **Inputs

X = Input(shpae=(Tx,n_values))** 

1. **1 .. Tx 루프**
    
    T에서의 x값은 (m,Tx,90) → (90,)로 슬라이스하여아함.
    
    Reshape x to be (1,n_values)
    
    LSTM_cell()
    
    densor()
    

```python
def djmodel(Tx, LSTM_cell, densor, reshaper):

	n_values = densor.units
	n_a = LSTM_cell.units
	
	X = Input(shape=(Tx, n_values))
	
	a0 = Input(shape=(n_a,), name='a0')
	c0 = Input(shape=(n_a,), name='c0')
	a = a0
	c = c0
	
	for t in range(Tx):
			x = X[:,t,:]
			x = reshaper(x)
			_, a, c = LSTM_cell(inputs=[x,a,c])
			out = densor(a)
			outputs.append(out)
	model = Model(inputs=[X, a0, c0], outputs=outputs)
	return model

model = djmodel(Tx=30, LSTM_cell=LSTM_cell, 
								densor=densor, reshaper=reshaper)

opt = Adam(lr=0.01, beta_1=0.9, beta_2=0.999, decay=0.01)

model.compile(optimizer=opt, loss='categorical_crossentropy', metrics=['accuracy'])
```

![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/085b3d2b-22d7-4f28-8d60-0d9139078e66)

> **음악 추론 모델**
> 

djmodel에서 전역객체로 생성한 각 layer들을 사용함으로써, 학습된 가중치를 활용하여, 학습을 진행한다.

```python
def music_inference_model(LSTM_cell, densor, Ty=100):
		# 배치사이즈를 지정해주지 않는다.
		x0 = Input(shape=(1, n_values))

		
		a0 = Input(shape=(n_a,), name='a0')
		c0 = Input(shape=(n_a,), name='c0')
		a = a0
		c = c0
		x = x0
		
		outputs = []
		
		for t in range(Ty):
					_, a, c = LSTM_cell(inputs=[x,a,c])
					out = densor(a)
					outputs.append(out)
					x = tf.math.argmax(out,axis=-1)
				  x = tf.one_hot(x, depth=n_values)
					# LSTM의 입력에 맞게 차원을 증가시켜주는 역할이다. (배치사이즈 차원증가)
					# (1, n_values) -> (1,1,n_values)
					x = RepeatVector(1)(x)
					
		
		inference_model = Model(inputs=[x0, a0, c0], outputs=outputs)
		return inference_model

inference_model = music_inference_model(LSTM_cell, densor, Ty = 50)

# Keras 프레임워크 상에서 x는 (m, Tx, n_values) 이며
# a, c는 (batchsize , unit개수)로 통일된다.
# 이모델의 배치가 1이므로 (1,1, n_values)이고 , (1,n_a)인것이다. 
x_initializer = np.zeros((1, 1, n_values))
a_initializer = np.zeros((1, n_a))
c_initializer = np.zeros((1, n_a))
```

90개의 시퀸스를 생성한다. (Ty) 가장큰 값을 사용하여, result에 담는다.

```python
def predict_and_sample(inference_model, x_initializer = x_initializer, a_initializer = a_initializer, 
                       c_initializer = c_initializer):

		n_values = x_initializer.shape[2]
		pred =inference_model.predict([x_initializer, a_initializer, c_initializer])
		indices = np.argmax(np.array(pred),axis=-1)
		results = to_categorical(indices, num_classes=n_values)
		return results, indices

results, indices = predict_and_sample(inference_model, x_initializer,
																					 a_initializer, c_initializer)

```

> **결과**
> 

[music.wav](/assets/music.wav)