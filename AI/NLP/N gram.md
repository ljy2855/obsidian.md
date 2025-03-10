
## Language Model

### Define
- 다음 단어가 나올 확률 모델(probalilstic model)
- 기존 단어들을 조합해서 다음 단어가 나올 확률을 계산함


#### Chain Rule
![[Pasted image 20250310165112.png]]

#### 그럼 N-gram이 뭔가

앞의 모든 단어를 모두 확인할 수 없으니 approximation을 통해 앞선 n개의 단어만 놓고 확인함

2개 보면 bigram
n개 보면 ngram
#### Markov assumption

앞선 단어 몇개만 보고 다음 나올 단어 추정

#### 단어 짱짱 많은데 어떻게 확률을 추정할 수 있을까
실제로


### Generation with n-gram language model

- 다음 나올 단어를 생성하는 방법은요?

1. 미리 확률 테이블을 만들어놓기
2. n grams을 넣고 가장 확률 높은 단어 넣기
3. 2 반복

- generation 방법
	- top-k vs top-p sampling
		- top-k : 상위 k개만 선택
		- top-p : 상위 top의 합이 일정 임계치가 되는 애들 선택
	![[Pasted image 20250310165720.png]]
long tail? -> top-k중 일부만 크고 나머진 구림 

### Evaluation

#### Extrinsic evaluation
- real-world task or downstream application
- 모델 자체보다 실제 어플리케이션이 얼마나 부합하는가 판단 지표
- Hard to optimize downstream objective (indirect feedback)

#### Perplexity (ppl) - Intrinsic evaluation
얼마나 다음 단어를 잘 예측하는가 지표

![[Pasted image 20250310170522.png]]




#### Intuition
