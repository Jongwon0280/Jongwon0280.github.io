---
layout: single
title:  "지역치안안전데이터 전처리"
categories:
  - da
  
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
author_profile: true
---

# 지역치안안전데이터분석경연대회

2월들어서 새로운 데이터 경연대회가 나왔다. 

경찰대학교에서 주관하는 지역치안안전데이터분석 경연대회이다. 

데이터분석경연대회를 꾸준히 참가했었는데, 주로 리더보드에서 public, private 스코어를 높이는 대회를 나가다보니 EDA 보다는 특징의 전처리, 특징선택에 집중하였었고, 데이터 형태에 적합한 모델이나 스태킹같은 기법을 사용하여 성능을 올렸던 것 같다.

이번대회는 모델을 통해 리더보드의 성적을 높이는 것이 아닌 경찰대학교에서 제공하는 치안데이터(대전 충남 지역)의 112신고데이터를 기반으로  주제의 인사이트 및 예측모델을 제안하는 것이다.

[스마트 치안 빅데이터 플랫폼](https://www.bigdata-policing.kr/board/b_contest/view?idx=167&category=)

### 공모주제

<aside>
🚗 1. 대전·세종·충남 지역 교통사고 분석 및 예측

</aside>

<aside>
🗣 2. 대전·세종·충남 지역 보이스피싱 분석 및 예측

</aside>

### 치안데이터분석이란?

<aside>
😀 빅데이터를 기반으로 한 치안 데이터 분석을 통해 범죄예방 및 대응, 치안 수요 예측 등 경찰 활동에 정보를 제공하고 관련 서비스 구축

</aside>

### 대회목적

- 미래 모빌리티와 연계한 지역 치안 안전 데이터 분석 경진대회를 통해 우수한 아이디어를 발굴·포상하고 지원하여 지역 치안 안전 확립
- 신종·지능형 범죄 급증 등 급변하는 치안 환경에 발맞춰 치안 데이터 활용, 한정된 경찰력을 선택과 집중에 따라 운영하여 선제적 대응과 시대적 여건에 부응하는 맞춤형 치안 서비스 제공

## 데이터 선정 및 전처리

데이터는 신청과 동시에 신청자의 메일로 보내주셨다. 샘플데이터가 따로 없고 주제를 선택하고 데이터를 받는 방식이기에, 데이터를 유추하면서 주제를 선정할 수 밖에 없었다. 

우리 팀은 112신고데이터에서 보이스피싱 데이터에 만약 신고내용이 포함되어있지 않다면, 대회를 진행하면서 외부 데이터를 결합하는데 있어서 무리가 있을 수 있고 다양한 측면에서 비용이 많이 들것 같다고 판단하였고, 1번 공모주제를 택하기로 정하였다. 

### 데이터 명세

| No. | ColumnName | DataType | Null | Comment |
| --- | --- | --- | --- | --- |
| 1 | RECV_DEPT_NM | VARCHAR2 (20 Byte) | N | 접수부서 코드 |
| 3 | RECV_CPLT_DM | VARCHAR2 (8 Byte) | Y | 접수완료일시 |
| 5 | NPA_CL | VARCHAR2 (2 Byte) | N | 경찰청구분[그룹코드: 32] |
| 6 | EVT_STAT_CD | VARCHAR2 (2 Byte) | Y | 사건상태코드 [그룹코드: 01: 사건상태] |
| 7 | EVT_CL_CD | VARCHAR2 (3 Byte) | Y | 사건종별코드 [그룹코드: 02] |
| 8 | RPTER_SEX | VARCHAR2 (1 Byte) | Y | 신고 성별 - 1: 남자  2: 여자  3: 불상 |
| 9 | HPPN_PNU_ADDR | VARCHAR2 (30 Byte) | Y | 발생지점(PNU) |
| 10 | HPPN_X | NUMBER (12,8) | Y | 발생좌표X |
| 11 | HPPN_Y | NUMBER (12,8) | Y | 발생좌표Y |
| 12 | SME_EVT_YN | VARCHAR2 (1 Byte) | Y | 동일사건여부 |

우리팀의 예상대로 데이터에는 사고분류. 즉 보이스피싱인지 교통사고인지는 구분 할 수 있지만, 디테일한 신고정보를 알 수 없었다. 또한 KP,NPA등으로 파일이 분리되어 있었기에 이를 병합해야했고 각 테이블의 형식이 맞지않아 형식을 맞춰주어야했다.

### 데이터 전처리 및 병합

<head>
  <style>
    table.dataframe {
      white-space: normal;
      width: 100%;
      height: 240px;
      display: block;
      overflow: auto;
      font-family: Arial, sans-serif;
      font-size: 0.9rem;
      line-height: 20px;
      text-align: center;
      border: 0px !important;
    }

    table.dataframe th {
      text-align: center;
      font-weight: bold;
      padding: 8px;
    }

    table.dataframe td {
      text-align: center;
      padding: 8px;
    }

    table.dataframe tr:hover {
      background: #b8d1f3; 
    }

    .output_prompt {
      overflow: auto;
      font-size: 0.9rem;
      line-height: 1.45;
      border-radius: 0.3rem;
      -webkit-overflow-scrolling: touch;
      padding: 0.8rem;
      margin-top: 0;
      margin-bottom: 15px;
      font: 1rem Consolas, "Liberation Mono", Menlo, Courier, monospace;
      color: $code-text-color;
      border: solid 1px $border-color;
      border-radius: 0.3rem;
      word-break: normal;
      white-space: pre;
    }

  .dataframe tbody tr th:only-of-type {
      vertical-align: middle;
  }

  .dataframe tbody tr th {
      vertical-align: top;
  }

  .dataframe thead th {
      text-align: center !important;
      padding: 8px;
  }

  .page__content p {
      margin: 0 0 0px !important;
  }

  .page__content p > strong {
    font-size: 0.8rem !important;
  }

  </style>
</head>



```python
pip install sklearn
```

<pre>
Looking in indexes: https://pypi.org/simple, https://us-python.pkg.dev/colab-wheels/public/simple/
Collecting sklearn
  Downloading sklearn-0.0.post1.tar.gz (3.6 kB)
  Preparing metadata (setup.py) ... [?25l[?25hdone
Building wheels for collected packages: sklearn
  Building wheel for sklearn (setup.py) ... [?25l[?25hdone
  Created wheel for sklearn: filename=sklearn-0.0.post1-py3-none-any.whl size=2344 sha256=16e08d9dcca1f1d7ce179f18c4b5e3f3ce30488be38e6ebfe03b52b8d276eee9
  Stored in directory: /root/.cache/pip/wheels/14/25/f7/1cc0956978ae479e75140219088deb7a36f60459df242b1a72
Successfully built sklearn
Installing collected packages: sklearn
Successfully installed sklearn-0.0.post1
</pre>

```python
pip install pandas
```

<pre>
Looking in indexes: https://pypi.org/simple, https://us-python.pkg.dev/colab-wheels/public/simple/
Requirement already satisfied: pandas in /usr/local/lib/python3.8/dist-packages (1.3.5)
Requirement already satisfied: python-dateutil>=2.7.3 in /usr/local/lib/python3.8/dist-packages (from pandas) (2.8.2)
Requirement already satisfied: numpy>=1.17.3 in /usr/local/lib/python3.8/dist-packages (from pandas) (1.21.6)
Requirement already satisfied: pytz>=2017.3 in /usr/local/lib/python3.8/dist-packages (from pandas) (2022.7)
Requirement already satisfied: six>=1.5 in /usr/local/lib/python3.8/dist-packages (from python-dateutil>=2.7.3->pandas) (1.15.0)
</pre>
# 라이브러리 호출



```python
import pandas as pd
import matplotlib as plt
import seaborn as sns
import numpy as np
```

# 데이터불러오기



```python
kp2020=pd.read_csv("/content/drive/MyDrive/dataset/LocalSafety_Dataset/Data/KP2020.csv",encoding='cp949')
kp2021=pd.read_csv("/content/drive/MyDrive/dataset/LocalSafety_Dataset/Data/KP2021.csv",encoding='cp949')

npa2020=pd.read_csv("/content/drive/MyDrive/dataset/LocalSafety_Dataset/Data/NPA2020.csv",encoding='cp949')
```

한글이 파일내에 포함되어 있기때문에 cp949로 인코딩해준다.



```python
kp2020.dtypes
```

<pre>
RECV_DEPT_NM      object
RECV_CPLT_DM      object
NPA_CL             int64
EVT_STAT_CD        int64
EVT_CL_CD          int64
RPTER_SEX        float64
HPPN_PNU_ADDR     object
HPPN_X           float64
HPPN_Y           float64
SME_EVT_YN        object
dtype: object
</pre>

```python
npa2020.dtypes
```

<pre>
RECV_CPLT_DT       int64
RECV_CPLT_TM       int64
NPA_CL             int64
EVT_STAT_CD        int64
EVT_CL_CD          int64
RPTER_SEX         object
HPPN_OLD_ADDR     object
HPPN_X           float64
HPPN_Y           float64
SME_EVT_YN        object
dtype: object
</pre>
kp의 접수일자와 시간은 obj타입이며, npa는 정수타입으로 되어있고

kp의 접수 성별의 타입은 정수이지만 npa는 obj타입이다.



```python
npa2020.head(5)
```

<pre>
   RECV_CPLT_DT  RECV_CPLT_TM  NPA_CL  EVT_STAT_CD  EVT_CL_CD RPTER_SEX  \
0      20200101             7      13           10        501         2   
1      20200101           132      13           10        501         1   
2      20200101            39      13           10        501         1   
3      20200101           110      13           10        601         3   
4      20200101           342      13           10        601         1   

             HPPN_OLD_ADDR      HPPN_X     HPPN_Y SME_EVT_YN  
0   대전광역시 중구 목동(행정:목동) 360  127.409270  36.333010          Y  
1  대전광역시 중구 대흥동(대흥동) 499-1  127.421295  36.325575        NaN  
2                      NaN  127.404663  36.341685        NaN  
3                      NaN    0.000000   0.000000        NaN  
4                      NaN  127.404663  36.341685        NaN  
</pre>

```python
kp2020.head(5)
```

<pre>
  RECV_DEPT_NM                 RECV_CPLT_DM  NPA_CL  EVT_STAT_CD  EVT_CL_CD  \
0          충남청  20/12/01 01:43:07.000000000      19           10        305   
1          대전청  20/12/01 02:05:04.000000000      13           10        601   
2          대전청  20/12/01 02:06:52.000000000      13           10        601   
3          충남청  20/12/01 02:37:25.000000000      19           10        606   
4          충남청  20/12/01 08:17:50.000000000      19           10        401   

   RPTER_SEX                   HPPN_PNU_ADDR      HPPN_X     HPPN_Y SME_EVT_YN  
0        1.0       충청남도 보령시 궁촌동(행정:대천4동) 369  126.598345  36.341537          Y  
1        3.0                             NaN  127.404663  36.341685        NaN  
2        1.0                             NaN  127.404663  36.341685        NaN  
3        3.0        충청남도 보령시 천북면 하만리 628-10   126.524980  36.474390          N  
4        2.0  충청남도 천안시 서북구 성정동(행정:성정2동) 1259  127.137160  36.826718        NaN  
</pre>

```python
kp2021.head(5)
```

<pre>
  RECV_DEPT_NM                 RECV_CPLT_DM  NPA_CL  EVT_STAT_CD  EVT_CL_CD  \
0          대전청  21/03/07 00:00:01.000000000      13           10        604   
1          대전청  21/03/07 00:02:13.000000000      13           10        201   
2          대전청  21/03/07 00:00:33.000000000      13           10        601   
3          대전청  21/03/07 00:01:18.000000000      13           10        601   
4          대전청  21/03/07 00:01:43.000000000      13           10        308   

   RPTER_SEX              HPPN_PNU_ADDR      HPPN_X     HPPN_Y SME_EVT_YN  
0        3.0          대전광역시 서구 둔산동 1122  127.373676  36.350975          Y  
1        1.0  대전광역시 유성구 상대동(원신흥동) 469-9  127.339018  36.347420        NaN  
2        3.0                        NaN  127.404663  36.341685        NaN  
3        3.0                        NaN  127.404663  36.341685        NaN  
4        1.0        대전광역시 중구 선화동 141-16  127.420455  36.330413        NaN  
</pre>

```python
kp2020.columns == kp2021.columns
```

<pre>
array([ True,  True,  True,  True,  True,  True,  True,  True,  True,
        True])
</pre>
## KPA 전처리


kp2020, kp2021은 동일한 데이터프레임형식을 가진다. 차이점은 접수일자가 다르다는 것이다.





또 npa2020 접수일자와 접수시간을 나타내는 컬럼이 다르기 때문에 전처리가 필요하다.  



```python
kp_merge = pd.concat([kp2020,kp2021], ignore_index=True)
```

kp_merge라는 데이터프레임으로 병합해주었다.



```python
kp_merge=kp_merge.drop('RECV_DEPT_NM',axis=1)
```

또한 kp에 접수부서코드를 삭제해주었다. 



**=> 우리 팀은 이 데이터에서 시간별 위경도 별 각 사건별 건수를 뽑아낼 것 이기때문에 접수부서 코드는 일시적으로 제외하는 걸로 하였다.**



```python
kp_merge.head()
```

<pre>
        RECV_CPLT_DM  NPA_CL  EVT_STAT_CD  EVT_CL_CD  RPTER_SEX  \
0  20/12/01 01:43:07      19           10        305        1.0   
1  20/12/01 02:05:04      13           10        601        3.0   
2  20/12/01 02:06:52      13           10        601        1.0   
3  20/12/01 02:37:25      19           10        606        3.0   
4  20/12/01 08:17:50      19           10        401        2.0   

                    HPPN_PNU_ADDR      HPPN_X     HPPN_Y SME_EVT_YN  \
0       충청남도 보령시 궁촌동(행정:대천4동) 369  126.598345  36.341537          Y   
1                             NaN  127.404663  36.341685        NaN   
2                             NaN  127.404663  36.341685        NaN   
3        충청남도 보령시 천북면 하만리 628-10   126.524980  36.474390          N   
4  충청남도 천안시 서북구 성정동(행정:성정2동) 1259  127.137160  36.826718        NaN   

  RECV_CPLT_TM RECV_CPLT_DT  
0     01:43:07     20/12/01  
1     02:05:04     20/12/01  
2     02:06:52     20/12/01  
3     02:37:25     20/12/01  
4     08:17:50     20/12/01  
</pre>

```python
kp_merge['RECV_CPLT_DM']=kp_merge['RECV_CPLT_DM'].str.slice(start=0, stop=17)
```


```python
kp_merge['RECV_CPLT_TM']=kp_merge['RECV_CPLT_DM'].str.slice(start=9)
```


```python
kp_merge['RECV_CPLT_DT']=kp_merge['RECV_CPLT_DM'].str.slice(start=0, stop=8)
```


```python
kp_merge = kp_merge.drop('RECV_CPLT_DM',axis=1)
```

kpa데이터에서 접수날짜와 접수시간은 공백을 구분자로 분리되어있기에, 이것을 슬라이싱하여 npa의 컬럼 "RECV_CPLT_DT" "RECV_CPLT_TM"으로 만들어주었다. 



```python
kp_merge.head()
```

<pre>
  RECV_CPLT_DT  RECV_YEAR  RECV_MONTH  RECV_DAY  RECV_HOUR  RECV_MINUTE  \
0   2020-12-01       2020          12         1          1           43   
1   2020-12-01       2020          12         1          2            5   
2   2020-12-01       2020          12         1          2            6   
3   2020-12-01       2020          12         1          2           37   
4   2020-12-01       2020          12         1          8           17   

   RECV_SEC  NPA_CL  EVT_STAT_CD  EVT_CL_CD RPTER_SEX  \
0         7      19           10        305       1.0   
1         4      13           10        601       3.0   
2        52      13           10        601       1.0   
3        25      19           10        606       3.0   
4        50      19           10        401       2.0   

                    HPPN_PNU_ADDR      HPPN_X     HPPN_Y SME_EVT_YN  
0       충청남도 보령시 궁촌동(행정:대천4동) 369  126.598345  36.341537          Y  
1                             NaN  127.404663  36.341685        NaN  
2                             NaN  127.404663  36.341685        NaN  
3        충청남도 보령시 천북면 하만리 628-10   126.524980  36.474390          N  
4  충청남도 천안시 서북구 성정동(행정:성정2동) 1259  127.137160  36.826718        NaN  
</pre>

```python
kp_merge['RECV_CPLT_TM']=pd.to_datetime(kp_merge['RECV_CPLT_TM'],format="%H:%M:%S")

kp_merge['RECV_HOUR']= kp_merge['RECV_CPLT_TM'].dt.hour
kp_merge['RECV_MINUTE']= kp_merge['RECV_CPLT_TM'].dt.minute
kp_merge['RECV_SEC']= kp_merge['RECV_CPLT_TM'].dt.second
```


```python
kp_merge.RECV_CPLT_DT=pd.to_datetime(kp_merge['RECV_CPLT_DT'],format="%y/%m/%d")

kp_merge['RECV_YEAR']= kp_merge['RECV_CPLT_DT'].dt.year
kp_merge['RECV_MONTH']= kp_merge['RECV_CPLT_DT'].dt.month
kp_merge['RECV_DAY']= kp_merge['RECV_CPLT_DT'].dt.day
```


```python
kp_merge=kp_merge.drop('RECV_CPLT_TM',axis=1)
```

접수일자 및 접수시간의 컬럼들을 datetime64으로 바꾸고 각 컬럼을 생성해주었다.(연도, 월, 일, 시간, 분 ,초)



```python
kp_merge.RPTER_SEX = kp_merge.RPTER_SEX.astype('str')
```

npa2020의 성별의 타입이 str이므로 동일하게 변경해주었다.



```python
npa2020.columns
```

<pre>
Index(['RECV_CPLT_DT', 'RECV_CPLT_TM', 'NPA_CL', 'EVT_STAT_CD', 'EVT_CL_CD',
       'RPTER_SEX', 'HPPN_OLD_ADDR', 'HPPN_X', 'HPPN_Y', 'SME_EVT_YN'],
      dtype='object')
</pre>

```python
kp_merge.columns
```

<pre>
Index(['NPA_CL', 'EVT_STAT_CD', 'EVT_CL_CD', 'RPTER_SEX', 'HPPN_PNU_ADDR',
       'HPPN_X', 'HPPN_Y', 'SME_EVT_YN', 'RECV_CPLT_DT', 'RECV_HOUR',
       'RECV_MINUTE', 'RECV_SEC', 'RECV_YEAR', 'RECV_MONTH', 'RECV_DAY'],
      dtype='object')
</pre>

```python
kp_merge=kp_merge[['RECV_CPLT_DT','RECV_YEAR', 'RECV_MONTH', 'RECV_DAY' ,'RECV_HOUR',
       'RECV_MINUTE', 'RECV_SEC','NPA_CL', 'EVT_STAT_CD', 'EVT_CL_CD', 'RPTER_SEX', 'HPPN_PNU_ADDR',
       'HPPN_X', 'HPPN_Y', 'SME_EVT_YN']]
```

npa의 컬럼순서대로 컬럼순서를 재배치하였다.


## NPA 전처리



```python
npa2020.columns
```

<pre>
Index(['RECV_CPLT_DT', 'RECV_CPLT_TM', 'NPA_CL', 'EVT_STAT_CD', 'EVT_CL_CD',
       'RPTER_SEX', 'HPPN_OLD_ADDR', 'HPPN_X', 'HPPN_Y', 'SME_EVT_YN'],
      dtype='object')
</pre>

```python
npa2020.dtypes
```

<pre>
RECV_CPLT_DT       int64
RECV_CPLT_TM       int64
NPA_CL             int64
EVT_STAT_CD        int64
EVT_CL_CD          int64
RPTER_SEX         object
HPPN_OLD_ADDR     object
HPPN_X           float64
HPPN_Y           float64
SME_EVT_YN        object
dtype: object
</pre>
NPA는 접수일자 및 접수시간이 int형으로 되어있었기에, obj형태로 바꾸어주어야한다.



```python
npa2020.RECV_CPLT_TM = npa2020.RECV_CPLT_TM.astype('str')
```


```python
npa2020[["RECV_CPLT_TM"]].head(10)
```

<pre>
  RECV_CPLT_TM
0            7
1          132
2           39
3          110
4          342
5          400
6          404
7          628
8          553
9          749
</pre>

```python
npa2020["RECV_CPLT_TM"]=npa2020["RECV_CPLT_TM"].str.zfill(6)
```

npa의 접수시간은 총6자리이며 6자리가 안되는 값들을 앞에 0으로 채워야한다.

       

       1728 => 00:17:28 => 0시17분28초를 의미한다.



```python
npa2020.RECV_CPLT_DT=pd.to_datetime(npa2020['RECV_CPLT_DT'],format="%Y%m%d")

npa2020['RECV_YEAR']= npa2020['RECV_CPLT_DT'].dt.year
npa2020['RECV_MONTH']= npa2020['RECV_CPLT_DT'].dt.month
npa2020['RECV_DAY']= npa2020['RECV_CPLT_DT'].dt.day
```


```python
npa2020['RECV_CPLT_TM']=pd.to_datetime(npa2020['RECV_CPLT_TM'],format="%H%M%S")

npa2020['RECV_HOUR']= npa2020['RECV_CPLT_TM'].dt.hour
npa2020['RECV_MINUTE']= npa2020['RECV_CPLT_TM'].dt.minute
npa2020['RECV_SEC']= npa2020['RECV_CPLT_TM'].dt.second
```


```python
npa2020=npa2020.drop('RECV_CPLT_TM',axis=1)
```

kp와 동일하게 접수날짜와 시간을 처리하였다.



```python
npa2020.rename(columns = {'HPPN_OLD_ADDR' : 'HPPN_PNU_ADDR'}, inplace = True)
```


```python
npa2020=npa2020[['RECV_CPLT_DT', 'RECV_YEAR', 'RECV_MONTH', 'RECV_DAY', 'RECV_HOUR',
       'RECV_MINUTE', 'RECV_SEC', 'NPA_CL', 'EVT_STAT_CD', 'EVT_CL_CD',
       'RPTER_SEX', 'HPPN_PNU_ADDR', 'HPPN_X', 'HPPN_Y', 'SME_EVT_YN']]
```


```python
npa2020.head(5)
```

<pre>
  RECV_CPLT_DT  RECV_YEAR  RECV_MONTH  RECV_DAY  RECV_HOUR  RECV_MINUTE  \
0   2020-01-01       2020           1         1          0            0   
1   2020-01-01       2020           1         1          0            1   
2   2020-01-01       2020           1         1          0            0   
3   2020-01-01       2020           1         1          0            1   
4   2020-01-01       2020           1         1          0            3   

   RECV_SEC  NPA_CL  EVT_STAT_CD  EVT_CL_CD RPTER_SEX  \
0         7      13           10        501         2   
1        32      13           10        501         1   
2        39      13           10        501         1   
3        10      13           10        601         3   
4        42      13           10        601         1   

             HPPN_PNU_ADDR      HPPN_X     HPPN_Y SME_EVT_YN  
0   대전광역시 중구 목동(행정:목동) 360  127.409270  36.333010          Y  
1  대전광역시 중구 대흥동(대흥동) 499-1  127.421295  36.325575        NaN  
2                      NaN  127.404663  36.341685        NaN  
3                      NaN    0.000000   0.000000        NaN  
4                      NaN  127.404663  36.341685        NaN  
</pre>
kp의 주소컬럼이름은 "PNU" 이므로 동일하게 "OLD" => "PNU"로 변경하고 kp와 동일한 순으로 컬럼을 재배치하였다. 



```python
merge_data = pd.concat([kp_merge,npa2020], ignore_index=True)
```


```python
merge_data
```

<pre>
        RECV_CPLT_DT  RECV_YEAR  RECV_MONTH  RECV_DAY  RECV_HOUR  RECV_MINUTE  \
0         2020-12-01       2020          12         1          1           43   
1         2020-12-01       2020          12         1          2            5   
2         2020-12-01       2020          12         1          2            6   
3         2020-12-01       2020          12         1          2           37   
4         2020-12-01       2020          12         1          8           17   
...              ...        ...         ...       ...        ...          ...   
3849376   2020-11-22       2020          11        22          0           35   
3849377   2020-11-22       2020          11        22          0           52   
3849378   2020-11-22       2020          11        22          0           46   
3849379   2020-11-22       2020          11        22          0           52   
3849380   2020-11-22       2020          11        22          0           11   

         RECV_SEC  NPA_CL  EVT_STAT_CD  EVT_CL_CD RPTER_SEX  \
0               7      19           10        305       1.0   
1               4      13           10        601       3.0   
2              52      13           10        601       1.0   
3              25      19           10        606       3.0   
4              50      19           10        401       2.0   
...           ...     ...          ...        ...       ...   
3849376         5      19           10        501         1   
3849377        13      13           10        601             
3849378        27      19           10        601         1   
3849379        46      19            5        301         1   
3849380        34      13           10        601             

                          HPPN_PNU_ADDR      HPPN_X     HPPN_Y SME_EVT_YN  
0             충청남도 보령시 궁촌동(행정:대천4동) 369  126.598345  36.341537          Y  
1                                   NaN  127.404663  36.341685        NaN  
2                                   NaN  127.404663  36.341685        NaN  
3              충청남도 보령시 천북면 하만리 628-10   126.524980  36.474390          N  
4        충청남도 천안시 서북구 성정동(행정:성정2동) 1259  127.137160  36.826718        NaN  
...                                 ...         ...        ...        ...  
3849376                                         NaN        NaN             
3849377                                  127.404663  36.341685             
3849378                                         NaN        NaN          Y  
3849379          충청남도 보령시 신흑동(행정:대천5동)   126.516040  36.305619             
3849380                                  127.404663  36.341685             

[3849381 rows x 15 columns]
</pre>
전처리한 kp_merge와 npa2020을 병합하였다.



```python
data=merge_data
```


```python
traffic_data = data[(data['EVT_CL_CD']>=401)&(data['EVT_CL_CD']<=406 ) ]
```





> 교통사고의 코드번호는 401-406이다.





![Untitled](/assets/localSafety/accident_type.png)  










> 401~406으로 조건기반 필터링을 진행.



```python
traffic_data=traffic_data.sort_values(['RECV_CPLT_DT','RECV_HOUR','RECV_MINUTE','RECV_SEC'])
```


```python
traffic_data.reset_index(drop=True,inplace=True)
```

> 날짜순으로 오름차순 정렬을 진행한다.



```python
traffic_data.isnull().sum()
```

<pre>
RECV_CPLT_DT          0
RECV_YEAR             0
RECV_MONTH            0
RECV_DAY              0
RECV_HOUR             0
RECV_MINUTE           0
RECV_SEC              0
NPA_CL                0
EVT_STAT_CD           0
EVT_CL_CD             0
RPTER_SEX             0
HPPN_PNU_ADDR      9556
HPPN_X             3827
HPPN_Y             3827
SME_EVT_YN       287616
dtype: int64
</pre>

```python
traffic_data.SME_EVT_YN=traffic_data.SME_EVT_YN.fillna('N')
```


```python
traffic_data['SME_EVT_YN'][traffic_data[traffic_data['SME_EVT_YN']==' '].index]='N'
```

> 많은 결측치를 차지하고있는 "SME_EVT_YN"은 동일 사건여부를 뜻하는 컬럼이므로 결측치는 동일사건이 아니다라고 결측을 처리 할 수 있다.



```python
traffic_data['RPTER_SEX'].unique()
```

<pre>
array(['1', '3', '2', '{', ' ', '1.0', '3.0', '2.0'], dtype=object)
</pre>
> 신고성별은 1(남) 2(여) 3(불상)으로 나뉘어야하므로 1.0 2.0 3.0을 형식을 통일해주고 "{" ," "은 불상으로 처리해준다.



```python
traffic_data['RPTER_SEX'][traffic_data[traffic_data['RPTER_SEX']=='{'].index]='3'
traffic_data['RPTER_SEX'][traffic_data[traffic_data['RPTER_SEX']==' '].index]='3'
traffic_data['RPTER_SEX'][traffic_data[traffic_data['RPTER_SEX']=='3.0'].index]='3'
traffic_data['RPTER_SEX'][traffic_data[traffic_data['RPTER_SEX']=='2.0'].index]='2'
traffic_data['RPTER_SEX'][traffic_data[traffic_data['RPTER_SEX']=='1.0'].index]='1'
```


```python
traffic_data.SME_EVT_YN.unique()
```

<pre>
array(['Y', 'N'], dtype=object)
</pre>

```python
traffic_data['RPTER_SEX'].unique()
```

<pre>
array(['1', '3', '2'], dtype=object)
</pre>