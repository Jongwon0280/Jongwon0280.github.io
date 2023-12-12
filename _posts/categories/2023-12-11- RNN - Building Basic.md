---
layout: single
title:  RNN - Building Basic
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-11
---

## RNN - Building Basic

****************Example : a5,(2)[3]<4>****************

→ 2번째 데이터 샘플, 3번째 레이어, 4번째 타임 시퀸스 5번째 엔트리이다.

 ********************************

> **Basic RNN**
> 

![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/07929a97-a2ea-4b7c-8926-2f37baf698ae)

예시에서는 시퀸스 길이인 Tx = Ty과 같다.

 

> **입력 X의 차원**
> 

1. **nx**는 유닛의 개수이다.
    
    x(i)<t>는 i번째 데이터샘플의 t번째 시퀸스의 입력을 의미한다. 
    
    언어를 예로들었을때 입력은 토큰화되어서 입력으로 들어가기 때문에, 원핫벡터로써 5000단어의 vocabulary사전이 있는 경우 (5000,)의 shape을 가진다. 즉 여기서는 5000이 nx가 된다.
    
2. **Tx**는 타임스텝의 사이즈이다.
    
    즉, x<1> ~ x<50>까지라면 타임스텝은 50
    
3. **m**은 배치사이즈이다.
    
    만약 20이 배치사이즈라고하면, 이를 벡터화시켜서, 쌓게 되고 결론적으로 (nx, m , Tx) 꼴로서 (5000, 20, 50)이 미니배치의 차원이 되는 것이다.
    

→ **결론적으로 위의 3차원 텐서가 미니배치로써 RNN의 입력으로 들어가게된다.**

1. 타임스텝별로 **2D 슬라이싱**을 취한다.
    
    미니배치별로 각 타임별로 슬라이싱을 해야하므로, step t마다, (nx, m)의 크기를 가지고 이는 x<t>로 사용된다.
    

> **은닉층 a의 차원**
> 

1. **na**는 유닛의 개수이다.
    
    만약 미니배치로 학습한다면 사이즈는 (na,m)이 될것이고, time차원을 포함한다면 (na, m, Tx)로 될것이다. 
    
    입력과 동일하게 루프를 통해 각 타임스텝 별 2d슬라이싱을 진행하고, a<t>형태로 활용한다.
    

> ****************************************************y^(출력값)의 차원****************************************************
> 

위의 은닉층과 입력층과 비슷하게 차원은 (ny,m,Ty)이다.

> **RNN Cell**
> 
[Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/1dad7fda-4465-4128-84d7-43a898250bad)

```python
def rnn_cell_forward(xt, a_prev, parameters):
	"""
	xt는 t에서의 x벡터 즉 (nx,m)
	a_prev 또한 동일하게 (na, m)
	parmeters는
				Wax => (na,nx)
				Waa => (na,na)
				Wya => (ny,na)
				ba => (na,1)
				by => (ny,1)
"""
a_next = np.tanh(np.dot(Waa,a_prev)+np.dot(Wax,xt)+ ba)
yt_pred = softmax(np.dot(Wya, a_next)+by)
cache = (a_next, a_prev, xt, parameters)
# cache는 backwarding할때 사용하는것.
return a_next, yt_pred, cache
```

> **RNN Forward Pass**
> 

만약 10times의 길이를 가지고있다면 위에서 정의한 RNN Cell를 10번을 사용해야한다.

![Untitled 2](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/156306a3-9727-4660-8e71-c8917033be0e)

각 cell의 필요한 입력으로는 다음과 같으면

- a<t-1>
- x<t>

각 cell의 출력으로는 다음과 같다

- a<t>
- y^<t>

**사용되는 가중치 행렬인 Waa, Wax, ba, by는 각 타임스텝마다 재활용된다.**

→ 학습의 효율성과 일반화 능력을 높이게 하기 위함이다.

> **********RNN_Forward Implements**********
> 

1. 모든 은닉층 a를 생성한다 (m개 TrainingSample)
    
    → shape : 3차원 텐서인 (na, m, Tx)
    
2. 출력을 저장할 y^을 생성한다.
    
    → shape : 3차원 텐서 (ny,m,Ty)
    
    이 예제에서는 Ty = Tx
    
3. a0 (초기 2차원텐서)를 시작으로 위에서 정의한 rnn_cell_forward를 통한 연산을 한다.

```python
def rnn_forward(x, a0, parameters):
	"""
	x -> (nx , m , Tx)
	a0 -> (na , m )
	parameters -> 가중치, 각 타임스텝마다 공유됨.
	
	"""
	caches=[]
	n_x, m, T_x = x.shape
  n_y, n_a = parameters["Wya"].shape

	
	a = np.zeros((n_a,m,T_x))
  y_pred = np.zeros((n_y,m,T_x))
	
	a_next = a0

	for t in range(T_x):
			a_next, yt_pred, cache = rnn_cell_forward(x[:,:,t],a_next,parameters)
			a[:,:,t] = a_next
			y_pred[:,:,t] = yt_pred
			caches.append(cache)
	
	caches = (caches, x)
    
  return a, y_pred, caches
```

> **Basic LSTM**
> 

![Untitled 3](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/02a53345-b9fd-4a1e-868a-3c9618258473)

1. **Forget gate** ****𝚪𝑓****
    
    역할은 문법구조를 추적하려고할때  예를들어 puppy or puppies에 따라 뒤에 사용되는 동사들이 달라져야하기때문에.
    
    forget gate는 0과1사이의 값을 포함한다.
    
    **공식 :** 
    
    𝚪⟨𝑡⟩𝑓=𝜎(𝐖𝑓[𝐚⟨𝑡−1⟩,𝐱⟨𝑡⟩]+𝐛𝑓)
    
    𝚪⟨𝑡⟩𝑓 * 𝐜⟨𝑡−1⟩ (요소곱이 가능하다. shape가 같아서)
    
2. **Candidate value** ****𝐜̃ ⟨𝑡⟩****
    
    현재 셀에 저장될 수 있는 시간단계의 정보를 포함하는 셀이다.
    
    전달되는 값은 update_gate에 의존한다. (-1 ~ +1)
    
    **공식:**
    
    𝐜̃ ⟨𝑡⟩=tanh(𝐖𝑐[𝐚⟨𝑡−1⟩,𝐱⟨𝑡⟩]+𝐛𝑐)
    
3. **Update Gate** ****𝚪𝑖****
    
    후보의 어떤 측면을 반영할지 결정하는 게이트이다. (0~1)
    
    0에가까워지면 전달이 되지 않는다.
    
    **공식:**
    
    𝚪⟨𝑡⟩𝑖=𝜎(𝐖𝑖[𝑎⟨𝑡−1⟩,𝐱⟨𝑡⟩]+𝐛𝑖)
    
    𝚪⟨𝑡⟩𝑖 * ****𝐜̃ ⟨𝑡⟩ (요소곱을 한후, c<t-1>에 합산한다.****
    
4. ****Cell state 𝐜⟨𝑡⟩****
    
    셀 상태는 미래로 전달되는 메모리이다.
    
    **공식 :**
    
    𝐜⟨𝑡⟩=𝚪⟨𝑡⟩𝑓∗𝐜⟨𝑡−1⟩+𝚪⟨𝑡⟩𝑖∗𝐜̃ ⟨𝑡⟩
    
    𝐜⟨𝑡−1⟩는 forget게이트에의해 조정되며,
    
    𝐜̃ ⟨𝑡⟩는 update게이트에 의해 조정된다.
    

1. **Output Gate** ****𝚪𝑜****
    
    출력에 관한 제어.
    
    **공식 :**
    
    𝚪⟨𝑡⟩𝑜=𝜎(𝐖𝑜[𝐚⟨𝑡−1⟩,𝐱⟨𝑡⟩]+𝐛𝑜)
    

1. **Hidden state** 𝐚⟨𝑡⟩
    
    다음 시간대의 forget, update, output를 결정하는데 사용된다.
    
    **공식 :**
    
    𝐚⟨𝑡⟩=𝚪⟨𝑡⟩𝑜∗tanh(𝐜⟨𝑡⟩)
    
2. **Prediction** ****𝐲⟨𝑡⟩𝑝𝑟𝑒d****
    
    예측을 위한 softmax
    
    **공식 :** 
    
    𝐲⟨𝑡⟩𝑝𝑟𝑒𝑑=softmax(𝐖𝑦𝐚⟨𝑡⟩+𝐛𝑦)
    

> **LSTM Cell**
> 

1. 연산 효율성을 위한 concat
    
    𝑐𝑜𝑛𝑐𝑎𝑡=[𝑎⟨𝑡−1⟩
    
               𝑥⟨𝑡⟩]
    
2. 위의 각 연산들을 구현
3. y^<t>를 계산

```python
def lstm_cell_forward(xt, a_prev, c_prev, parameters):
	"""
	matrix
	xt : (n_x, m)
	a_prev : (n_a,m)
	c_prev : (n_a,m)
	parameters : RNN에서 개별로 되어있던 가중치들을 concat하여 관리
	Wf -- (n_a, n_a + n_x)
	ex) np.dot(Wfa , A<t-1>) + np.dot(Wfx, X<t>) + bf 
  bf -- (n_a, 1)
  Wi -- (n_a, n_a + n_x)
  bi -- (n_a, 1)
  Wc -- (n_a, n_a + n_x)
  bc -- (n_a, 1)
  Wo -- (n_a, n_a + n_x)
  bo -- (n_a, 1)
  Wy -- (n_y, n_a)
  by -- (n_y, 1)
	
	"""
	n_x, m = xt.shape
  n_y, n_a = Wy.shape

	concat = np.concatenate([a_prev,xt],axis=0) # 수평쌓기

	ft = sigmoid(np.dot(Wf,concat)+bf)
  it = sigmoid(np.dot(Wi,concat)+bi)
  cct = np.tanh(np.dot(Wc, concat)+bc)
  c_next = ft * c_prev + it * cct
  ot = sigmoid(np.dot(Wo, concat)+bo)
  a_next = ot * np.tanh(c_next)

	yt_pred = softmax(np.dot(Wy, a_next)+by)

	cache = (a_next, c_next, a_prev, c_prev, ft, it, cct, ot, xt, parameters)

  return a_next, c_next, yt_pred, cache
```

> **LSTM Forward Pass**
> 

![Untitled 4](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/89edd516-bf3f-4c66-a5e4-59f3eb30927f)

1. 3차원 텐서 a,c,y를 초기화한다.
    
    a : (na, m, Tx)
    
    c: (na , m, Tx)
    
    y: (ny, m, Tx)
    
2. a<t>는 a0로 초기화한다.
3. c<t>는 0벡터로 초기화한다.
4. 3차원 a,c,y를 2차원 텐서로 슬라이싱하여 위에서 구현한 lstm_cell_forward를 통해 계산한다.

```python
def lstm_forward(x, a0, parameters):
	caches = []
	a = np.zeros((n_a,m,T_x))
  c = np.zeros((n_a,m,T_x))
  y = np.zeros((n_y,m,T_x))
	
	a_next = a0
    
  c_next = np.zeros((n_a,m))
	
	for t in range(T_x):
			xt = x[:,:,t]
			a_next, c_next, yt, cache = lstm_cell_forward(xt, a_next, c_next, parameters)
			a[:,:,t] = a_next
			c[:,:,t]  = c_next
			y[:,:,t] = yt
			caches.append(cache)
	caches = (caches, x)
	return a, y, c, caches
```

> **RNN vs LSTM**
> 

![Untitled](RNN%20-%20Building%20Basic%20e632f93805334437bb93d73e007a2e16/Untitled%201.png)

![Untitled](RNN%20-%20Building%20Basic%20e632f93805334437bb93d73e007a2e16/Untitled%203.png)

RNN의 고질적인 문제점은 기울기 소멸과 폭발문제이다.

"I am studying _____ because I want to become a machine learning engineer.”

이 문장에서 machine learning engineer라는 단어는 빈칸을 예측하는데 큰 역할을 할 수 있지만, 만약 **기울기 소멸문제**로 인해 지역적으로 멀리 떨어진 machine learning engineer의 가중치가 전파가 되지 않는다면, 예측을 하는데 문제가 생긴다. 

- 반복적인 구조
    
    RNN은 셀마다 동일한 연산을 진행하는데 있어서, 이전 셀상태와 가중치들의 행렬 곱연산이 많고, 활성함수인 tanh로 인하여 기울기 폭발, 기울기 소멸문제에 기여한다.
    

이점을 해결하기 위한 LSTM은 게이트라는 것을 도입하여 문제를 해결한다.

![Untitled](RNN%20-%20Building%20Basic%20e632f93805334437bb93d73e007a2e16/Untitled%203.png)

요약하면 c<t>인 기억저장소는 이전 시퀸스에서의 장기적인 메모리를 입력하며, 현재 셀에서의 입력을 반영하여 저장하고, a<t>는 입력과 이전 셀상태를 반영하여 출력을 만들며, 현재 입력 정보를 더 반영한다.

<aside>
💡 𝚪⟨𝑡⟩𝑓=𝜎(𝐖𝑓[𝐚⟨𝑡−1⟩,𝐱⟨𝑡⟩]+𝐛𝑓) → **망각회로**

</aside>

입력을 반영하여 c<t-1>의 특정위치의 값이 보존되어야하는지 삭제되어야하는지를 결정한다.

활성함수가 sigmoid이기때문에, 0~1로 출력이 나오며 이를 요소곱을 진행한다.

<aside>
💡 𝚪⟨𝑡⟩𝑖=𝜎(𝐖𝑖[𝑎⟨𝑡−1⟩,𝐱⟨𝑡⟩]+𝐛𝑖) → **업데이트**

𝐜̃ ⟨𝑡⟩=tanh(𝐖𝑐[𝐚⟨𝑡−1⟩,𝐱⟨𝑡⟩]+𝐛𝑐) → **후보 셀상태**

𝐜⟨𝑡⟩=𝚪⟨𝑡⟩𝑓∗𝐜⟨𝑡−1⟩+𝚪⟨𝑡⟩𝑖∗𝐜̃ ⟨𝑡⟩ → **셀 상태**

</aside>

새로운 시퀸스의 입력을 얼마나 메모리에 반영할지를 결정하는 파트이며, 후보셀상태로 생성한 정보를 얼마나 반영할것인지를 update gate를 통해 (0~1)로 구하여 요소곱을 한후, 이를 c<t>에 합산함으로써, c<t>는 망각회로를 통해 과거정보를 얼마나 가져갈것인지와, 수정회로와 입력회로를 통해 새로운 시퀸스의 정보를 얼마나 반영할것인지를 구한다.

<aside>
💡 𝚪⟨𝑡⟩𝑜=𝜎(𝐖𝑜[𝐚⟨𝑡−1⟩,𝐱⟨𝑡⟩]+𝐛𝑜) → **출력회로**

𝐚⟨𝑡⟩=𝚪⟨𝑡⟩𝑜∗tanh(𝐜⟨𝑡⟩) → **은닉층**

𝐲⟨𝑡⟩𝑝𝑟𝑒𝑑=softmax(𝐖𝑦𝐚⟨𝑡⟩+𝐛𝑦) → **예측 회로**

</aside>

a<t>는 현재의 내부 메모리 상태와 현재 입력을 고려한 LSTM 셀의 최종 출력이며, output게이트(0~1)는 위에서 결합한 c<t> ( 망각회로, 업데이트, 입력을 반영한)에서 얼마만큼을 cell의 최종출력에 반영할것인지를 결정하는 것이며, 이를 요소곱을 함으로써, 정보가 필터링된다.

**결론적으로는, RNN과 달리 C<t>라는 장기의존정보저장소의 선형적인 연산의 결합을 통해 이전 C<t-1>과의 값의 차이가 별로없으므로, 기울기 흐름에 장애물을 만들지 않기때문에, 기울기가 크게 감소하거나 증가하는 흐름을 만들지 않는다.** 

> **추가정보**
> 

→ 이는 앞선 이미지모델의 ResNet의 잔차학습과 비슷한 해결책이다.

→ GRU는 LSTM의 경량화 버전이다.

→ 프레임워크 사용시, m은 무시해서 구현해야한다.