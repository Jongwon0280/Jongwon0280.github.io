---
layout: single
title:  RNN - Character Level Lang Model
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-11
---

## RNN - Character Level Lang Model

> **목적**
> 

RNN을 활용한 공룡의 이름 붙이기

> **활용**
> 
- RNN의 텍스트데이터 저장
- RNN을 사용한 문자수준의 텍스트 생성모델구축
- RNN에서 새로운 시퀸스 샘플링
- Gradient Clipping

> **데이터셋**
> 
- 19909개의 공룡이름.txt
- 어휘사전의 크기는 알파벳 26+\n=27이다.

```python
#어휘사전만들기 (a -> 1 , 1-> a)
char_to_ix = { ch:i for i,ch in enumerate(chars) }
ix_to_char = { i:ch for i,ch in enumerate(chars) }
```

> **모델**
> 

![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/dd98d8c8-7de0-4b5b-abb7-f9af98889b22)

이 문제의 핵심은 **다음글자를 예측하는것이므로, 항상 y<t>=x<t+1>**가 된다.

> **클리핑**
> 

기울기 폭발문제를 방지하기 위해 임계점을 주는것.

![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/f5ee7656-bf67-4b55-b4da-ee84494eb8aa)

```python
def clip(gradients, maxValue):

		gradients = copy.deepcopy(gradients)
    
    dWaa, dWax, dWya, db, dby = gradients['dWaa'], gradients['dWax'], gradients['dWya'], gradients['db'], gradients['dby']
		
		for gradient in [dWaa,dWax,dWya,db,dby]:
        np.clip(gradient, (-1)*maxValue,maxValue, out =gradient )
		
		gradients = {"dWaa": dWaa, "dWax": dWax, "dWya": dWya, "db": db, "dby": dby}
    
    return gradients
		
```

> **샘플링**
> 

![Untitled 2](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/ca025bba-4894-46f0-b587-d3886892c225)

1. **a<0>,  x<1>를 0벡터**로 초기화한다.
2. **a<1>, y^<1>**를 구한다.
    
    ![Untitled 3](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/0439dc99-9dcf-4499-885a-efbbff211ae4)
    
    **y^<t+1>는 다음에 올수있는 문자 27개의 각 확률(0~1)을 나타낸다.**
    
3. **샘플링**
    
    softmax에서 가장높은 확률을 가진 글자를 사용하면 주어진 시작문자에 대해 항상 동일한 결과가 생성되고, 이름생성에 있어서 다양한 결과를 위한 해결책으로 np.random.choice를 활용하여 다음문자를 선택한다.
    
    ```python
    def sample(parameters, char_to_ix, seed):
    	"""
    		Waa(100, 100)
    		Wax(100, 27)
    		Wya(27, 100)
    		by(27, 1)
    		b(100, 1)
    	"""
    	Waa, Wax, Wya, by, b = parameters['Waa'], parameters['Wax'], parameters['Wya'], parameters['by'], parameters['b']
      vocab_size = by.shape[0]
      n_a = Waa.shape[1]
    	
    	x = np.zeros((vocab_size,1))  
      a_prev = np.zeros((n_a,1))
    	
    	indices = []
    	idx = -1
    	
    	counter = 0
      newline_character = char_to_ix['\n']
    
    	while (idx != newline_character and counter != 50):
          
          a = np.tanh(np.dot(Wax, x)+np.dot(Waa,a_prev)+b)
          z = np.dot(Wya, a)+by
          y = softmax(z) #(27,1)
    
          np.random.seed(counter + seed) 
          
          idx = np.random.choice(range(len(y.ravel())), p = y.ravel()) #(27,)
    			indices.append(idx)
    			
    			# 샘플링한 결과의 인덱스를 새로운 x<t+1>로반영
    			x = np.zeros((vocab_size, 1))
          x[idx] = 1
    			
    			a_prev = a
    			
    			seed += 1
          counter +=1
    
    			#50자제한
    			if (counter == 50):
            indices.append(char_to_ix['\n'])
    	return indices
    ```
    
    > **언어 모델 구축**
    > 
    
    ```python
    def model(data_x, ix_to_char, char_to_ix, num_iterations = 35000, n_a = 50, dino_names = 7, vocab_size = 27, verbose = False):
    	n_x, n_y = vocab_size, vocab_size
    
    	#공룡이름 \n를 기준으로 나누어서 리스트로 저장
    	examples = [x.strip() for x in data_x]
    	
    	np.random.seed(0)
      np.random.shuffle(examples)
    
    	a_prev = np.zeros((n_a, 1))
    	last_dino_name = "abc"
    
    	for j in range(num_iterations):
    			idx = j % (len(examples))
    
    			# 공룡이름 알파벳으로 분리 및 인덱스화
    			single_example = examples[idx]
          single_example_chars = [ c for c in single_example]
          single_example_ix = [char_to_ix[ch] for ch in single_example_chars]
    			X = [None] + single_example_ix # 첫시퀸스는 0벡터추가
    			
    			ix_newline = char_to_ix['\n']
          Y = X[1:] + [ix_newline] # Y[t] = x[t+1]이므로 1부터 마지막에는 개행을 붙여준다.
    	
    			curr_loss, gradients, a_prev = optimize(X, Y, a_prev, parameters, learning_rate=0.01)
    			
    			if j % 2000 == 0:
                
                print('Iteration: %d, Loss: %f' % (j, loss) + '\n')
                
                # The number of dinosaur names to print
                seed = 0
                for name in range(dino_names): #7개
                    
                    # Sample indices and print them
                    sampled_indices = sample(parameters, char_to_ix, seed)
                    last_dino_name = get_sample(sampled_indices, ix_to_char)
                    print(last_dino_name.replace('\n', ''))
                    
                    seed += 1  # To get the same result (for grading purposes), increment the seed by one. 
          
                print('\n')
            
        return parameters, last_dino_name
    ```
    
    반복을거듭할수록 사우루스라는 이름을 부여하는 것을 확인할 수 있다.
    
    ```
    Iteration: 0, Loss: 23.087336
    
    Nkzxwtdmfqoeyhsqwasjkjvu
    Kneb
    Kzxwtdmfqoeyhsqwasjkjvu
    Neb
    Zxwtdmfqoeyhsqwasjkjvu
    Eb
    Xwtdmfqoeyhsqwasjkjvu
    
    ....
    
    Iteration: 22000, Loss: 22.728886
    
    Onustreofkelus
    Llecagosaurus
    Mystolojmiaterltasaurus
    Ola
    Yuskeolongus
    Eiacosaurus
    Trodonosaurus
    ```