# Pandas 라이브러리 기초 사용법 정리

```python
import pandas as pd
df = pd.read_csv(friend_src, encoding='utf-8') # friend_src는 사용하려는 csv의 위치를 지정해줌
df.head() # 앞 5개의 데이터 출력 
df.to_csv(new_friend_src, encoding='utf-8') # to_csv를 통해 해당 위치에 데이터프레임을 csv로 저장 가능


pd.Series([1,2,3])
# 0  1
# 0  2
# 0  3
# dtype: int64    왼쪽과 같이 나오게됨


series = df['name']
# 0     John
# 1    Jenny
# 2     Nate
# 3    Julia
# 4    Brian
# 5    Chris
# Name: name, dtype: object


series = pd.Series([10,2,5,4], index=['a','b','c','d'], dtype=float)
# a    10.0
# b     2.0
# c     5.0
# d     4.0
# dtype: float64
series.sort_values(ascending=False) # 이렇게 정렬을 할 수도 있다. 기본값은 오름차순임.  


series = pd.Series([10,2,5,4], index=['a','d','c','b'], dtype=float)
series.sort_index(ascending=False) # index를 가지고 정렬도 가능하다.  


df = pd.DataFrame({'a':[2,3], 'b':[5,10]})
#   a	b
# 0	2	5
# 1	3	10
df = pd.DataFrame([[2,5],[3,10],[4,15]], columns=['a','b'])
#   a	b
# 0	2	5
# 1	3	10
# 2	4	15
df = pd.DataFrame([[2,5],[3,10],[4,15]], columns=['a','b'], index['가','나','다'],dtype=float)
#   a	b
# 가	2	5
# 나	3	10
# 다	4	15
```

<img width="249" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/a6984630-dc8c-471e-9556-744c736efdef">

이제부터 위의 데이터프레임을 예시로 사용한다.  

```python
df.iloc[0]
# name       John
# age          20
# job     student
# Name: 0, dtype: object

df.iloc[[0]]       # [0]은 Series 형태로, []로 한 번 더 묶어주면 DataFrame으로 가져온다.
#   name	age	job
# 0	John	20	student


df.iloc[0:2, 0:2]  # 이와 같이 슬라이싱을 이용할 수가 있다.
# 	name	age
# 0	John	20
# 1	Jenny	30

df.iloc[:,[1,2]]	# 슬라이싱을 사용하지 않고 특정 행/열만 가져올 수도 있다.
# 	b	c
# 0	1	2
# 1	5	6
# 2	9	10

df['job']
# 0      student
# 1    developer
# 2      teacher
# 3      dentist
# 4      manager
# 5       intern
# Name: job, dtype: object


df.loc[:3,'job']  # loc는 인덱스와 컬럼을 문자로, iloc는 리스트 배열로 선택한다.
# 0      student
# 1    developer
# 2      teacher
# 3      dentist
# Name: job, dtype: object

df.loc[[0], 'job']
# 0    student
# Name: job, dtype: object


df = pd.DataFrame(np.arange(12).reshape(3,4), columns=['a','b','c','d'])
# 	a	b	c	d
# 0	0	1	2	3
# 1	4	5	6	7
# 2	8	9	1011

df.drop([1,2], axis=0) # 행의 인덱스 1,2에 해당하는 행 삭제, axis = 0이 default
#   a	b	c	d
# 1	4	5	6	7

df.drop(['a','b'], axis=1) # 컬럼'a','b'에 해당하는 열 삭제
#   c	d
# 0	2	3
# 1	6	7
# 2	1011


df.loc[1, 'c']=55	# 인덱스1, 컬럼 'c'에 해당하는 위치에 55로 초기화
df.iloc[1,2]=6		# 2행 3열의 위치에 6으로 초기화
df.iloc[:,1]=10		# 2열 모두 10으로 초기화
df.loc[:,:]=10		# 모두 10으로 초기화


# 다시 위에서 보았던 name, age, job 데이터프레임을 가지고 진행
df[[True, True, True,True,True,False]]  # False인 행은 가져오지 않는다.

df[(df['age']>20) & (df['age']<31)]  # 이렇게 조건을 내부에 넣어 조건에 맞는 행만 가져오게 할 수 있다.  
#   name	age	job
# 1	Jenny	30	developer
# 2	Nate	30	teacher
# 5	Chris	25	intern

df[(df['age']>=40) | (df['age']<30)] # or의 경우 '|' 로 처리
df[df['job'].apply(lambda x: x in ['student', 'manager'])] # 이렇게 처리할 수도 있음
#   name	age	job
# 0	John	20	student
# 4	Brian	45	manager

```

<br/>

<img width="1069" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/8c47e175-fbac-4fe2-9136-46333a4be668">

이제 위의 데이터를 이용하여 진행한다.  

```python
abalone_src=base_src+'/abalone.data'
abalone_df=pd.read_csv(abalone_src, header=None, sep=',', names=['sex','length','diameter','height', 
                  'whole_weight','shucked_weight','viscera_weight', 
                  'shell_weight','rings'])

abalone_df.shape
# (4177, 9)

abalone_df.isnull().sum()
# sex               0
# length            0
# diameter          0
# height            0
# whole_weight      0
# shucked_weight    0
# viscera_weight    0
# shell_weight      0
# rings             0
# dtype: int64

abalone_df.describe() # describe를 입력하면 count, mean, min 등 각종 수치정보가 나온다

abalone_df.groupby('sex').mean()
#	length	diameter	height	whole_weight	shucked_weight	viscera_weight	shell_weight	rings
# sex								
# F	0.579093	0.454732	0.158011	1.046532	0.446188	0.230689	0.302010	11.129304
# I	0.427746	0.326494	0.107996	0.431363	0.191035	0.092010	0.128182	7.890462
# M	0.561391	0.439287	0.151381	0.991459	0.432946	0.215545	0.281969	10.705497

abalone_df['length_bool']=np.where(abalone_df['length']>abalone_df['length'].median(), 
                                   'length_long', 'length_short')
# length 컬럼이 length의 중간값보다 넘으면 'length_long', 짧으면 'length_short'로 데이터프레임에 'length_boo'이라는 컬럼으로 데이터를 추가함

abalone_df.groupby(['sex', 'length_bool']).mean() # 2개의 그룹으로 나눠서 중간값 출력

abalone_df.duplicated().sum() # 중복 행의 수를 알려준다

new_abalone_df= new_abalone_df.drop_duplicates() # 중복을 drop

nan_abalone_df = abalone_df.copy() # 데이터프레임 복사

nan_abalone_df.loc[2, 'length']=np.nan # 결측치를 넣고 싶을 때에는 np.nan 이용

zero_abalone_df=nan_abalone_df.fillna(0) # 결측치를 채울 때에는 fillna를 이용

abalone_df[['diameter', 'whole_weight']].apply(np.average, axis=0)
# diameter와 whole_weight의 평균을 계산 axis가 0일 때에는 행을 합쳐 계산, 1일 때에는 열 합쳐 계산  
# diameter        0.407881
# whole_weight    0.828742
# dtype: float64               이런 식으로 나오게된다. 1로 하면 각 행별 저 2개의 값의 평균이 다 출력이된다.

import math
def avg_ceil(x,y,z):
  return math.ceil((x+y+z))

abalone_df[['diameter','height', 'whole_weight']].apply(lambda x: avg_ceil(x[0], x[1], x[2]), axis=0)
# 이렇게 함수를 생성해두고 이용도 가능
```

---

