---
layout: single
title:  Transformers Architecture with TensorFlow
categories:
  - ML
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-12-11
---

## Transformers Architecture with TensorFlow

> **목적**
> 

트랜스포머 Seq2Seq 아키텍처를 이해하고, 직접 tensorflow로 구현하는 목적이다.

> **필요성**
> 

Seq2Seq는 Transformer 이전에, Bi-LSTM기반 인코더와 LSTM의 인코더에서 attention를 적용하여, 해결하였다. 하지만 이 방법 역시, 게이트기반 RNN구조로 장기의존성을 해결하는 구조지만 완전히 해결하지 못하며, 기울기 소멸문제의 여지가 있다. 그래서 나온 Transformer는 LSTM이라는 모듈 자체를 없애고, 이를 Self-Attention이라는 기법, 즉 Attention만으로 인코딩한다.

> **구현**
> 

- **라이브러리**
    
    ```python
    # Tensorflow 프레임워크를 사용한다.
    
    import tensorflow as tf
    import time
    import numpy as np
    import matplotlib.pyplot as plt
    
    from tensorflow.keras.layers import Embedding, MultiHeadAttention, Dense, Input, Dropout, LayerNormalization
    from transformers import DistilBertTokenizerFast #, TFDistilBertModel
    from transformers import TFDistilBertForTokenClassification # pytorch에서는 BertForSequenceClassification
    ```
    
- ****Positional Encoding****
    
    RNN기반의 LSTM에서는 순차적인 모델의 특성상 위치정보를 따로 줄 필요가 없지만, 트랜스포머에서는 모든 시퀸스를 동등하게 입력해주기때문에, 순서정보를 잃는다. 이를 위해 Pos Encoding이 필요하다.
    
    ![Untitled](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/e91a9814-a413-4285-8f81-61b9f4222ff1)
    
    d는 단어임베딩의 차원을 뜻하고, pos는 단어의 위치, i = k//2
    
    cos과 sin을 사용하는 이유는 단순 하드코딩을 통해 위치인코딩을 하면 의미론적의미가 왜곡 되기에, 왜곡되지않고, 값이 작은 cos,sin을 이용한다.
    
    ![Untitled 1](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/8797193a-1b8b-47c9-9281-c8be65c61252)
    
    - **get angles**
        
        ```python
        def get_angles(pos, k, d):
            """
            Get the angles for the positional encoding
            
            Arguments:
                pos -- Column vector containing the positions [[0], [1], ...,[N-1]]
                k --   Row vector containing the dimension span [[0, 1, 2, ..., d-1]]
                d(integer) -- Encoding size
            
            Returns:
                angles -- (pos, d) numpy array 
            """
            
            
            # Get i from dimension span k
            i = k//2
            # 여기서 시퀸스 위치를 담는 pos 열벡터가 브로드캐스팅 되어
            # 각 시퀸스 위치에 대해 i인덱스에 대해 연산한다.
            angles = pos / (10000**(2*i/d))
          
            
            return angles
        ```
        
    - **Pos Encoding**
        
        ```python
        
        def positional_encoding(positions, d):
            """
            Precomputes a matrix with all the positional encodings 
            
            Arguments:
                positions (int) -- Maximum number of positions to be encoded 
                d (int) -- Encoding size 
            
            Returns:
                pos_encoding -- (1, position, d_model) A matrix with the positional encodings
            """
            
            angle_rads = get_angles(np.arange(positions)[:, np.newaxis], # 열벡터
                                    np.arange(d)[np.newaxis, :], # 행벡터
                                    d)
          
            # 짝수는 sin
            angle_rads[:, 0::2] = np.sin(angle_rads[:, 0::2])
          
            # 홀수는 cos
            angle_rads[:, 1::2] = np.cos(angle_rads[:, 1::2])
            
            
            pos_encoding = angle_rads[np.newaxis, ...]
            
            return tf.cast(pos_encoding, dtype=tf.float32)
        
        pos_encoding = positional_encoding(50, 512)
        ```
        
        ![Untitled 2](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/5ca50347-4d0f-47b5-b6af-a0053d6131bf)
        
- **마스킹**
    
    
    - **패딩마스크**
        
        
        입력 시퀸스의 최대길이는 정해져있지만, 각각 다른 시퀸스길이를 가지기 때문에, 길이가 짧은 토큰들은 패딩이 필요하다.
        
        <aside>
        🗽
        
        ```
        [["Do", "you", "know", "when", "Jane", "is", "going", "to", "visit", "Africa"],
         ["Jane", "visits", "Africa", "in", "September" ],
         ["Exciting", "!"]
        ]
        
        ```
        
        ```
        [[ 71, 121, 4, 56, 99, 2344, 345, 1284, 15],
         [ 56, 1285, 15, 181, 545],
         [ 87, 600]
        ]
        ```
        
        ```
        [[ 71, 121, 4, 56, 99],
         [ 2344, 345, 1284, 15, 0],
         [ 56, 1285, 15, 181, 545],
         [ 87, 600, 0, 0, 0],
        ]
        ```
        
        </aside>
        
        ```python
        def create_padding_mask(decoder_token_ids):
            """
            Creates a matrix mask for the padding cells
            
            Arguments:
                decoder_token_ids -- (n, m) matrix
            
            Returns:
                mask -- (n, 1, m) binary tensor
            """    
            seq = 1 - tf.cast(tf.math.equal(decoder_token_ids, 0), tf.float32)
          
            # add extra dimensions to add the padding
            # to the attention logits. 
            # this will allow for broadcasting later when comparing sequences
            return seq[:, tf.newaxis, :]
        ```
        
        주의할점은 0에대한 소프트맥스가 영향력을 발휘하지 않게 하기위해서는, (1-x) * -1.0e9)해주면 0으로 수렴하게 할 수 있다.
        
        ```python
        print(tf.keras.activations.softmax(x))
        print(tf.keras.activations.softmax(x + (1 - create_padding_mask(x)) * -1.0e9))
        
        """
        tf.Tensor(
        [[7.2876644e-01 2.6809821e-01 6.6454901e-04 6.6454901e-04 1.8064314e-03]
         [8.4437378e-02 2.2952460e-01 6.2391251e-01 3.1062774e-02 3.1062774e-02]
         [4.8541026e-03 4.8541026e-03 4.8541026e-03 2.6502505e-01 7.2041273e-01]], shape=(3, 5), dtype=float32)
        tf.Tensor(
        [[[7.2973627e-01 2.6845497e-01 0.0000000e+00 0.0000000e+00 1.8088354e-03]
          [2.4472848e-01 6.6524094e-01 0.0000000e+00 0.0000000e+00 9.0030573e-02]
          [6.6483547e-03 6.6483547e-03 0.0000000e+00 0.0000000e+00 9.8670328e-01]]
        
         [[7.3057163e-01 2.6876229e-01 6.6619506e-04 0.0000000e+00 0.0000000e+00]
          [9.0030573e-02 2.4472848e-01 6.6524094e-01 0.0000000e+00 0.0000000e+00]
          [3.3333334e-01 3.3333334e-01 3.3333334e-01 0.0000000e+00 0.0000000e+00]]
        
         [[0.0000000e+00 0.0000000e+00 0.0000000e+00 2.6894143e-01 7.3105860e-01]
          [0.0000000e+00 0.0000000e+00 0.0000000e+00 5.0000000e-01 5.0000000e-01]
          [0.0000000e+00 0.0000000e+00 0.0000000e+00 2.6894143e-01 7.3105860e-01]]], shape=(3, 3, 5), dtype=float32)
        """
        ```
        
    
    - **Look-Ahead Mask**
        
        디코더 단계에서 masked-MHA를 사용할때 필요한 마스크이다.
        
        ```python
        def create_look_ahead_mask(sequence_length):
            """
            Returns a lower triangular matrix filled with ones
            
            Arguments:
                sequence_length -- matrix size
            
            Returns:
                mask -- (size, size) tensor
            """
            mask = tf.linalg.band_part(tf.ones((1, sequence_length, sequence_length)), -1, 0)
            return mask
        ```
        
    
- **Self-Attention**
    
    
    ![Untitled 3](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/b804d620-1eaa-4c36-b360-eb1c0ae2df7d)
    
    최대 시퀸스 길이안의 모든 토큰들을 Wq, Wk, Wv (공유되는 파라미터)를 통해 고정된 임베딩 차원 q, k, v형태로 변환한다. 이후 각 시퀸스의 query는 모든 시퀸스 key와의 내적을 통해 각시퀸스별로 len(k)의 값을 얻어, (seq_q, seq_k) 행렬을 얻고 이를 softmax에 통과시켜 확률분포로 변환한다. 이후 각 시퀸스별로 v를 내적하여 각 시퀸스별로 더한다. 결과적으로 (시퀸스길이, 임베딩차원)의 결과값이 나온다.
    
    ![Untitled 4](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/80744435-d53b-4998-9fe0-0ba870283be5)
    
    ```python
    # UNQ_C3 (UNIQUE CELL IDENTIFIER, DO NOT EDIT)
    # GRADED FUNCTION scaled_dot_product_attention
    def scaled_dot_product_attention(q, k, v, mask):
        """
        Calculate the attention weights.
          q, k, v must have matching leading dimensions.
          k, v must have matching penultimate dimension, i.e.: seq_len_k = seq_len_v.
          The mask has different shapes depending on its type(padding or look ahead) 
          but it must be broadcastable for addition.
    
        Arguments:
            q -- query shape == (..., seq_len_q, depth)
            k -- key shape == (..., seq_len_k, depth)
            v -- value shape == (..., seq_len_v, depth_v)
            mask: Float tensor with shape broadcastable 
                  to (..., seq_len_q, seq_len_k). Defaults to None.
    
        Returns:
            output -- attention_weights
        """
       
        
        matmul_qk = tf.matmul(q,k,transpose_b=True)  # (..., seq_len_q, seq_len_k)
    
        # scale matmul_qk
        
        dk = float(q.shape[-1])
        scaled_attention_logits = matmul_qk / tf.math.sqrt(dk)
    
        # softmax에서 패딩마스크 0에 해당하는 위치의 영향력을 없애기 위함이다.
        if mask is not None: 
            scaled_attention_logits += ((1. - mask) * -1e9) 
    
        # 각 시퀸스의 쿼리에 대해 모든 시퀸스와의 키와의 내적에 대해서 softmax를 취해 확률 분포로 바꾼다.
      
        attention_weights = tf.keras.activations.softmax(scaled_attention_logits,axis=-1)  # (..., seq_len_q, seq_len_k)
    
        output = tf.matmul(attention_weights, v)  
    		# 각 시퀸스의 query에대한 각 시퀸스의 key에 대한 attention score를 각 시퀸스에 대해 구할 때마다 모든 시퀸스-
    		#에 value를 내적해줘야 하므로 가중합을 수행한다.
        
       
    
        return output, attention_weights
    ```
    
- **Encoder**
    
    ![Untitled 5](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/01b4092e-1938-43f0-9b7a-2b3940f9b3e4)
    
    멀티헤드 어텐션이란, 셀프 어텐션을 head의 수만큼 수행하여, concat하고, 이를 특정차원으로 임베딩시킨다. 각 셀프어텐션에서는 서로 다른 문맥의 특징을 캡처할 수 있다.
    
    특정차원으로 임베딩 시킨 벡터에 skip-connection을 더해주고, Layer 정규화를 수행시킨 후 FFNN의 입력으로 넣어주고 똑같이 skip-connection과 Layer 정규화를 진행시킨다.
    
    이과정은 Encoder Block의 1회 수행시 하는 일이고, 이를 k번수행한다.
    
     다음 코드는 인코더 블록의 FFNN을 정의한것이다.
    
    ```python
    def FullyConnected(embedding_dim, fully_connected_dim):
        return tf.keras.Sequential([
            tf.keras.layers.Dense(fully_connected_dim, activation='relu'),  # (batch_size, seq_len, dff)
            tf.keras.layers.Dense(embedding_dim)  # (batch_size, seq_len, embedding_dim)
        ])
    ```
    
    인코더 레이어를 정의하면 다음과 같다.
    
    ```python
    
    class EncoderLayer(tf.keras.layers.Layer):
       
        def __init__(self, embedding_dim, num_heads, fully_connected_dim,
                     dropout_rate=0.1, layernorm_eps=1e-6):
            super(EncoderLayer, self).__init__()
    
            self.mha = MultiHeadAttention(num_heads=num_heads,
                                          key_dim=embedding_dim,
                                          dropout=dropout_rate)
    
            self.ffn = FullyConnected(embedding_dim=embedding_dim,
                                      fully_connected_dim=fully_connected_dim)
    
            self.layernorm1 = LayerNormalization(epsilon=layernorm_eps)
            self.layernorm2 = LayerNormalization(epsilon=layernorm_eps)
    
            self.dropout_ffn = Dropout(dropout_rate)
        
        def call(self, x, training, mask):
            """
            
            
            Arguments:
                x -- Tensor of shape (batch_size, input_seq_len, embedding_dim)
                training -- Boolean, set to true to activate
                            the training mode for dropout layers
                mask -- Boolean mask to ensure that the padding is not 
                        treated as part of the input
            Returns:
                encoder_layer_out -- Tensor of shape (batch_size, input_seq_len, embedding_dim)
            """
            
           
            # Dropout is added by Keras automatically if the dropout parameter is non-zero during training
            self_mha_output = self.mha(x,x,x,mask) # Wq, Wk, Wv에 연산할 x, x, x와, 패딩 마스크를 입력으로 넣어준다.
    				# Self attention (batch_size, input_seq_len, embedding_dim)
            
            # skip connection
     
            skip_x_attention =  self.layernorm1(self_mha_output + x) # (batch_size, input_seq_len, embedding_dim)
    
            
            ffn_output = self.ffn(skip_x_attention)  # (batch_size, input_seq_len, embedding_dim)
            
            
            ffn_output = self.dropout_ffn(ffn_output,training=training)
            
       
            encoder_layer_out = self.layernorm2(skip_x_attention + ffn_output)  # (batch_size, input_seq_len, embedding_dim)
            
            
            return encoder_layer_out
    ```
    
- **Full Enocder**
    
    Encoder Layer를 k번 수행한다. 이를 반영한 Encoder Class를 정의하자.
    
    ```python
    
    class Encoder(tf.keras.layers.Layer):
    	  """
    		Encoder layer를 쌓는다.
    		"""
        def __init__(self, num_layers, embedding_dim, num_heads, fully_connected_dim, input_vocab_size,
                   maximum_position_encoding, dropout_rate=0.1, layernorm_eps=1e-6):
            super(Encoder, self).__init__()
    
            self.embedding_dim = embedding_dim
            self.num_layers = num_layers
    
            self.embedding = Embedding(input_vocab_size, self.embedding_dim)
            self.pos_encoding = positional_encoding(maximum_position_encoding, 
                                                    self.embedding_dim)
    
            self.enc_layers = [EncoderLayer(embedding_dim=self.embedding_dim,
                                            num_heads=num_heads,
                                            fully_connected_dim=fully_connected_dim,
                                            dropout_rate=dropout_rate,
                                            layernorm_eps=layernorm_eps) 
                               for _ in range(self.num_layers)]
    
            self.dropout = Dropout(dropout_rate)
            
        def call(self, x, training, mask):
            """
            
            
            Arguments:
                x -- Tensor of shape (batch_size, input_seq_len)
                training -- Boolean, set to true to activate
                            the training mode for dropout layers
                mask -- Boolean mask to ensure that the padding is not 
                        treated as part of the input
            Returns:
                out2 -- Tensor of shape (batch_size, input_seq_len, embedding_dim)
            """
            seq_len = tf.shape(x)[1]
            
            
            # Pass input through the Embedding layer
            x = self.embedding(x)  # (batch_size, input_seq_len, embedding_dim)
            # Scale embedding by multiplying it by the square root of the embedding dimension
            x *= tf.math.sqrt(tf.cast(self.embedding_dim, tf.float32))
            # 임베딩의 제곱근을 곱하는 이유는 (Q K^T) / (root(dk))로 나누어, 스케일링 해주기 위함이다.
            x += self.pos_encoding [:, :seq_len, :]
            # 초기화 단계에서 계산한, (1, input_seq_len , embedding dims)를 더해준다.
            # use `training=training`
            x = self.dropout(x,training=training)
            # 레이어의 수 만큼 encoder layer를 수행한다. 유동적으로 조절 가능하다. 
            for i in range(self.num_layers):
                x = self.enc_layers[i](x, training, mask) # 첫 인코더에는 입력으로 x
    						# 이후에는 encoder layer의 출력을 x로 입력하기 위함이다.
            
    
            return x  # (batch_size, input_seq_len, embedding_dim)
    ```
    
- **Decoder**
    
    
    ![Untitled 6](https://github.com/Jongwon0280/Jongwon0280.github.io/assets/56438131/5bbef429-44a8-4fbc-8ae7-de84581c3cfe)
    
    디코더는 인코더의 출력을 활용하여 seq2seq를 수행하기 위한 구조이다.
    
    인코더와 마찬가지의 구조이지만, look-ahead마스크를 활용한 masked-multi head attention과, enc_output의 key,value와 masked-multi head attention의 query를 활용하여 multi-head-attention을 수행하고, 시퀸스 한개씩 예측하고, 예측한 시퀸스의 값을 decoder의 입력으로 다시 활용한다. 
    
    다음은 디코더 레이어에 대한 구현이다.
    
    ```python
    
    class DecoderLayer(tf.keras.layers.Layer):
       
        def __init__(self, embedding_dim, num_heads, fully_connected_dim, dropout_rate=0.1, layernorm_eps=1e-6):
            super(DecoderLayer, self).__init__()
    
            self.mha1 = MultiHeadAttention(num_heads=num_heads,
                                          key_dim=embedding_dim,
                                          dropout=dropout_rate)
    
            self.mha2 = MultiHeadAttention(num_heads=num_heads,
                                          key_dim=embedding_dim,
                                          dropout=dropout_rate)
    
            self.ffn = FullyConnected(embedding_dim=embedding_dim,
                                      fully_connected_dim=fully_connected_dim)
    
            self.layernorm1 = LayerNormalization(epsilon=layernorm_eps)
            self.layernorm2 = LayerNormalization(epsilon=layernorm_eps)
            self.layernorm3 = LayerNormalization(epsilon=layernorm_eps)
    
            self.dropout_ffn = Dropout(dropout_rate)
        
        def call(self, x, enc_output, training, look_ahead_mask, padding_mask):
            """
            
            
            Arguments:
                x -- Tensor of shape (batch_size, target_seq_len, embedding_dim)
                enc_output --  Tensor of shape(batch_size, input_seq_len, embedding_dim)
                training -- Boolean, set to true to activate
                            the training mode for dropout layers
                look_ahead_mask -- Boolean mask for the target_input
                padding_mask -- Boolean mask for the second multihead attention layer
            Returns:
                out3 -- Tensor of shape (batch_size, target_seq_len, embedding_dim)
                attn_weights_block1 -- Tensor of shape(batch_size, num_heads, target_seq_len, input_seq_len)
                attn_weights_block2 -- Tensor of shape(batch_size, num_heads, target_seq_len, input_seq_len)
            """
            
            
            # enc_output.shape == (batch_size, input_seq_len, embedding_dim)
            
            # BLOCK 1
          
    				# look-ahead mask를 통해 masking된 마스크를 넣어준다. Q,K,V로 계산될 x,x,x는 인코더와 동일하다.
            mult_attn_out1, attn_weights_block1 = self.mha1(x, x, x, look_ahead_mask, return_attention_scores=True)  # (batch_size, target_seq_len, embedding_dim)
           
            Q1 = self.layernorm1(x+mult_attn_out1)
    
            # BLOCK 2
            
    				# Q에는 이전 디코더 Masked-MHA의 출력을, K,V는 사전에 계산한 Encoder의 output을 사용하고, 패딩마스크를 넣어준다.
            mult_attn_out2, attn_weights_block2 = self.mha2(Q1,enc_output, enc_output, padding_mask, return_attention_scores=True)  # (batch_size, target_seq_len, embedding_dim)
            
            
            mult_attn_out2 = self.layernorm2(mult_attn_out2 + Q1)  # (batch_size, target_seq_len, embedding_dim)
                    
            #BLOCK 3
            # pass the output of the second block through a ffn
            ffn_output = self.ffn(mult_attn_out2)  # (batch_size, target_seq_len, embedding_dim)
            
           
            ffn_output = self.dropout_ffn(ffn_output)
            
           
            out3 = self.layernorm3(ffn_output +mult_attn_out2 )  # (batch_size, target_seq_len, embedding_dim)
            
    
            return out3, attn_weights_block1, attn_weights_block2
    ```
    
- **Full Decoder**
    
    이젠 전체 디코더 구조에대한 클래스 구현이다.
    
    ```python
    
    class Decoder(tf.keras.layers.Layer):
    
        def __init__(self, num_layers, embedding_dim, num_heads, fully_connected_dim, target_vocab_size,
                   maximum_position_encoding, dropout_rate=0.1, layernorm_eps=1e-6):
            super(Decoder, self).__init__()
    
            self.embedding_dim = embedding_dim
            self.num_layers = num_layers
    
            self.embedding = Embedding(target_vocab_size, self.embedding_dim)
            self.pos_encoding = positional_encoding(maximum_position_encoding, self.embedding_dim)
    
            self.dec_layers = [DecoderLayer(embedding_dim=self.embedding_dim,
                                            num_heads=num_heads,
                                            fully_connected_dim=fully_connected_dim,
                                            dropout_rate=dropout_rate,
                                            layernorm_eps=layernorm_eps) 
                               for _ in range(self.num_layers)]
            self.dropout = Dropout(dropout_rate)
        
        def call(self, x, enc_output, training, 
               look_ahead_mask, padding_mask):
            """
            
            
            Arguments:
                x -- Tensor of shape (batch_size, target_seq_len, embedding_dim)
                enc_output --  Tensor of shape(batch_size, input_seq_len, embedding_dim)
                training -- Boolean, set to true to activate
                            the training mode for dropout layers
                look_ahead_mask -- Boolean mask for the target_input
                padding_mask -- Boolean mask for the second multihead attention layer
            Returns:
                x -- Tensor of shape (batch_size, target_seq_len, embedding_dim)
                attention_weights - Dictionary of tensors containing all the attention weights
                                    each of shape Tensor of shape (batch_size, num_heads, target_seq_len, input_seq_len)
            """
    
            seq_len = tf.shape(x)[1]
            attention_weights = {}
            
            
            # 워드 임베딩을 생성한다.
            x = self.embedding(x)  # (batch_size, target_seq_len, embedding_dim)
            
            # 스케일링을 위한 root( 임베딩차원)으로 곱해준다.
            x *= tf.math.sqrt(tf.cast(self.embedding_dim, tf.float32))
            
            # 초기화 단계에서 계산한 pos encoding을 더해줘, 위치 정보를 살린다.
            x += self.pos_encoding [:, :seq_len, :]
    
            x = self.dropout(x,training=training)
    
            
            for i in range(self.num_layers):
                
                x, block1, block2 = self.dec_layers[i](x, enc_output, training, look_ahead_mask, padding_mask)
    
                
                attention_weights['decoder_layer{}_block1_self_att'.format(i+1)] = block1
                attention_weights['decoder_layer{}_block2_decenc_att'.format(i+1)] = block2
            
            
           
            return x, attention_weights
    ```