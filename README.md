# LDA_test
```python
import warnings      # 경고 무시
warnings.filterwarnings('ignore')

import pandas as pd   
import numpy as np
from tqdm import tqdm  # 진행률 확인 라이브러리
import ast  # 문자열(str) -> 리스트(list)


import re     # 정규식 -> 텍스트 전처리를 위함

from operator import itemgetter  # 딕셔너리 정렬을 위함 

from kiwipiepy import Kiwi       # 형태소 분석기 
from collections import Counter  # item 빈도수 


# LDA 실행
from gensim import corpora
import gensim
import pyLDAvis.gensim_models  # LDA 시각화

```

# 데이터 불러오기

## 뉴스데이터

['본문1', '본문2', '본문3', ......]


```python
file_path = './data/test_news_data.txt'

with open(file_path,'r', encoding='utf-8-sig') as file:
    n_bodies = file.read()
```


```python
n_bodies = ast.literal_eval(n_bodies)
n_bodies[0]
```


## 불용어 정의

- 불용어 : 의미가 없는 단어들 <br>
 -->후에 해당 단어들은 삭제해주기 위해 불용어를 미리 정의 

키워드<br>
키워드2<br>
키워드3<br>
...


```python
# 불용어사전 1 _ 불용어 사전
file = open('data/stopwords.txt', encoding='utf8')
stopwords = file.read()
stopwords = stopwords.split('\n')
file.close()
```

# 기사 - 형태소 분석

## 형태소 분석 준비

### 형태소 분석 대상 전처리 함수

- 전처리: 데이터를 분석에 적합한 형태로 가공하는 과정
- 텍스트 전처리는 데이터 특성상 노이즈(ex.특수기호, 불용)가 굉장히 많음 --> 전처리 필요


```python
def cleansingTextforStemming(text) :
    text = text.replace('(주)', "").replace('주식회사', '').replace('(',' ').replace(')',' ')
    text = re.sub('[\xa0]',' ', text)
    text = re.sub('[=+#\?:^$@*\"※~%!』\\‘|\(\)\[\]\<\>`\'…》▶◆▲ⓒ©△◇■©▲↓【】;]', ' ', text)
    text = re.sub('[\n\t/·ㆍ∙&-]', ' ', text)
    text = re.sub('[^ㄱ-ㅎㅏ-ㅣ가-힣.a-zA-Z 0-9, ]', '', text)
    
    # 영어 -> 대문자 변환
#     text = text.upper()

    return text
```

### 실행 함수


```python
def stemming(text):
    
    # 영어 대문자 변환
    text = text.upper()
    tokens = []
    # 형태소 분석
    kiwi_stems = kiwi.analyze(text)
    stem_list = kiwi_stems[0][0]
#     print(kiwi_stems)
    for stem in stem_list:
        tokens.append(stem[0])
    
    return tokens
#    return NNs
```

## 실행


```python
kiwi = Kiwi()
kiwi.prepare()
```




    0




```python
token_docs = []  # 빈리스트 생성
for n_body in tqdm( n_bodies ):
    
    if n_body != None: 
        text = cleansingTextforStemming(n_body)   # 텍스트 전처리
    else :
        text = ''
    
    
    corp_kws = stemming(text)     # 형태소 분석
    clean_kws = [ kw for kw in corp_kws if (kw not in stopwords) and (len(kw) > 1)]  # 불용어 제거 + 한글자 키워드 제거
    
    token_docs.append(clean_kws)
    

```

    100%|████████████████████████████████████████████████████████████████████████████| 44596/44596 [35:50<00:00, 20.74it/s]


# LDA 실행

- https://wikidocs.net/30708

## LDA를 위한 데이터 형태 만들기


```python
dictionary = corpora.Dictionary(token_docs)     # 딕셔너리화
corpus = [dictionary.doc2bow(text) for text in token_docs]   # 문서 데이터 -> 수치화
```


```python
print(corpus[1], '\n\n' , dictionary[0], len(dictionary))
```

## Topic Num 찾기
- findTopicNum_simple.ipynb , FindTopicNum_결과확인 파일에서 실행

1. topic : 당신이 가설로 잡은 토픽의 갯수
2. chunksize : 얼마나 많은 문서가 훈련 알고리즘에 사용되는가?
    - 만약에 빠른 학습이 중요하시다면, 청크사이즈를 키워서 돌려보기
    - Hoffman의 논문에 의하면 Chunksize는 모델 품질에 영향을 미치지만 차이그 그렇게 크진 않다고 함
3. passes : 패스는 모델 학습시 전체 코퍼스에서 모델을 학습시키는 빈도를 제어
    - epochs 와 같은 용어
    - model를 학습시키는 횟수
4. iteration : 각각 문서에 대해서 루프를 얼마나 돌리는지를 제어
    - pass & iteration 은 최대한 많은게 좋음
5. eval_every = 1 in LdaModel
6. alpha, eta = auto, 디리클레 분포의 감마함수에 대한 파라미터
7. id2word: Dictionary 에 list of list of str 형식의 documents 를 입력하면 Dictioanry 가 학습됩니다.

**파라미터 튜닝 참고 : https://coredottoday.github.io/2018/09/17/%EB%AA%A8%EB%8D%B8-%ED%8C%8C%EB%9D%BC%EB%AF%B8%ED%84%B0-%ED%8A%9C%EB%8B%9D/**

## LDA 실행


```python
NUM_TOPICS = 15
ldamodel = gensim.models.ldamodel.LdaModel(corpus, num_topics = NUM_TOPICS, id2word=dictionary, passes=10, random_state=777)
# topics = ldamodel.print_topics(num_words=4)

pyLDAvis.enable_notebook()
vis = pyLDAvis.gensim.prepare(ldamodel, corpus, dictionary)
```


```python
pyLDAvis.display(vis)
```


### 토픽별 키워드 확인


```python
topics = ldamodel.print_topics(num_words=20)
for topic in topics:
    print(topic)
    print('\n')
```

    (38, '0.030*"콘텐츠" + 0.014*"공간" + 0.013*"제작" + 0.013*"영상" + 0.012*"디자인" + 0.011*"코로나" + 0.010*"채널" + 0.009*"유튜브" + 0.009*"전시" + 0.009*"브랜드" + 0.009*"여행" + 0.009*"온라인" + 0.008*"크리에이터" + 0.008*"작품" + 0.007*"방송" + 0.007*"기획" + 0.006*"라이브" + 0.006*"호텔" + 0.006*"인스타그램" + 0.005*"작가"')
    
    
    (36, '0.087*"브랜드" + 0.049*"브랜드평판" + 0.041*"평판지수" + 0.031*"빅데이터" + 0.023*"상장기업" + 0.017*"하락" + 0.017*"소통지수" + 0.017*"커뮤니티지수" + 0.017*"참여지수" + 0.016*"한국기업" + 0.015*"미디어지수" + 0.014*"기업평판" + 0.011*"헬릭스미스" + 0.011*"사회공헌" + 0.011*"전기자전거" + 0.011*"평판연구소" + 0.011*"시장지수" + 0.009*"공헌지수" + 0.008*"통신장비" + 0.008*"치매치료제"')
    
    
    (31, '0.014*"문제" + 0.009*"대통령" + 0.008*"변호사" + 0.008*"규제" + 0.008*"주장" + 0.007*"피해" + 0.006*"경우" + 0.006*"사실" + 0.006*"국회" + 0.005*"조사" + 0.005*"신규등록" + 0.005*"외식" + 0.005*"논란" + 0.005*"제기" + 0.005*"발표" + 0.005*"경제" + 0.005*"호반건설" + 0.004*"조치" + 0.004*"사회" + 0.004*"정책"')
    
    
    (8, '0.044*"친환경" + 0.028*"태양광" + 0.023*"환경" + 0.020*"에너지" + 0.016*"생산" + 0.015*"사회" + 0.015*"플라스틱" + 0.013*"재활용" + 0.012*"재생에너지" + 0.011*"신재생" + 0.011*"소재" + 0.008*"지속가능" + 0.007*"산업훈장" + 0.007*"공장" + 0.007*"종합화학" + 0.007*"선박" + 0.006*"폐기물" + 0.006*"탄소" + 0.006*"공급" + 0.006*"탄소중립"')
    
    
    (14, '0.015*"온라인" + 0.012*"정보" + 0.012*"마케팅" + 0.011*"네이버" + 0.010*"결제" + 0.009*"광고" + 0.009*"거래" + 0.009*"모바일" + 0.008*"배송" + 0.008*"판매" + 0.007*"시스템" + 0.007*"배달" + 0.007*"디지털" + 0.007*"비대면" + 0.007*"주문" + 0.006*"데이터" + 0.006*"오프라인" + 0.006*"코로나" + 0.006*"지역" + 0.006*"구독"')
    
    
    (6, '0.085*"기사" + 0.071*"분류" + 0.036*"구독" + 0.033*"섹션" + 0.028*"제보" + 0.019*"전재" + 0.019*"무단" + 0.015*"스포츠" + 0.013*"언론사" + 0.013*"정보" + 0.013*"가이드" + 0.013*"안내" + 0.012*"자동" + 0.012*"시스템" + 0.011*"이드" + 0.011*"종목" + 0.010*"네이버스포츠" + 0.009*"제외" + 0.009*"기사제공" + 0.009*"취소"')
    
    
    (1, '0.086*"포스코" + 0.034*"최정우" + 0.026*"GS리테일" + 0.020*"상품권" + 0.013*"롯데" + 0.012*"배달앱" + 0.011*"포스코그룹" + 0.011*"마이크로니" + 0.010*"막걸리" + 0.010*"롯데그룹" + 0.010*"한화그룹" + 0.009*"배민" + 0.008*"광산업" + 0.008*"롯데쇼핑" + 0.008*"포스코케미칼" + 0.008*"한화" + 0.008*"킨텍스" + 0.007*"새벽배송" + 0.007*"태영건설" + 0.007*"나라장터"')
    
    
    (4, '0.079*"게임" + 0.015*"블록체인" + 0.015*"모바일" + 0.014*"캐릭터" + 0.012*"이벤트" + 0.011*"콘텐츠" + 0.009*"플레이" + 0.008*"아이템" + 0.007*"텐센트" + 0.007*"가상자산" + 0.006*"컴투스" + 0.006*"시스템" + 0.006*"그래픽" + 0.006*"넥슨" + 0.006*"영웅" + 0.005*"게임즈" + 0.005*"거래소" + 0.005*"비트코인" + 0.005*"신작" + 0.004*"제작"')
    
    
    (23, '0.074*"경우" + 0.071*"회원" + 0.022*"정보" + 0.022*"약관" + 0.020*"행위" + 0.018*"회원가입" + 0.017*"책임" + 0.011*"개인정보" + 0.011*"동의" + 0.009*"규정" + 0.009*"저작" + 0.008*"게시" + 0.008*"게재" + 0.008*"중지" + 0.007*"비밀번호" + 0.007*"아이디" + 0.007*"손해" + 0.007*"공지" + 0.007*"위반" + 0.006*"침해"')
    
    
    (24, '0.034*"반도체" + 0.028*"배터리" + 0.023*"삼성전자" + 0.019*"소재" + 0.018*"생산" + 0.015*"장비" + 0.015*"정답" + 0.013*"부품" + 0.012*"디스플레이" + 0.011*"운동" + 0.011*"스마트폰" + 0.011*"제조" + 0.011*"공급" + 0.011*"삼성" + 0.009*"퀴즈" + 0.009*"센서" + 0.009*"리브메이" + 0.008*"공장" + 0.008*"공정" + 0.008*"모듈"')
    
    
    (13, '0.023*"자율주행" + 0.022*"차량" + 0.017*"자동차" + 0.014*"안전" + 0.014*"모빌리티" + 0.011*"전기차" + 0.009*"산업단지" + 0.009*"지역" + 0.008*"양극재" + 0.007*"이동" + 0.007*"인프라" + 0.007*"충전" + 0.006*"시스템" + 0.006*"첨단기술" + 0.006*"택시" + 0.006*"모델" + 0.005*"운전자" + 0.005*"실증" + 0.005*"업무협약" + 0.005*"설치"')
    
    
    (37, '0.018*"식품" + 0.016*"식물" + 0.013*"음식" + 0.013*"메뉴" + 0.012*"맥주" + 0.011*"고기" + 0.011*"와인" + 0.010*"건강" + 0.010*"브랜드" + 0.009*"주류" + 0.009*"음료" + 0.008*"재료" + 0.008*"배달" + 0.008*"샐러드" + 0.008*"판매" + 0.008*"커피" + 0.007*"단백질" + 0.007*"요리" + 0.007*"박스" + 0.007*"편의점"')
    
    
    (7, '0.071*"금융" + 0.032*"베트남" + 0.031*"대출" + 0.029*"은행" + 0.022*"교육지원" + 0.020*"마이데이터" + 0.020*"금리" + 0.017*"핀테크" + 0.013*"금융위원회" + 0.012*"신용" + 0.011*"금융지주" + 0.010*"금융기관" + 0.010*"캐피탈" + 0.010*"신용평가" + 0.008*"발급" + 0.008*"금융서비스" + 0.008*"우리은행" + 0.008*"국민은행" + 0.008*"신용대출" + 0.007*"금융회사"')
    
    
    (11, '0.041*"화장품" + 0.041*"피부" + 0.039*"여성" + 0.024*"성분" + 0.021*"뷰티" + 0.018*"건강" + 0.016*"정원" + 0.012*"콜라겐" + 0.011*"재배" + 0.011*"원료" + 0.010*"함유" + 0.010*"남성" + 0.008*"브랜드" + 0.008*"농업" + 0.007*"케어" + 0.005*"크림" + 0.005*"LG생활건강" + 0.005*"품종" + 0.005*"수분" + 0.005*"안전"')
    
    
    (39, '0.030*"사람" + 0.027*"생각" + 0.014*"교육" + 0.011*"정도" + 0.011*"학생" + 0.011*"문제" + 0.008*"강의" + 0.008*"대학" + 0.007*"이야기" + 0.007*"지금" + 0.006*"경우" + 0.006*"설명" + 0.006*"온라인" + 0.006*"교수" + 0.005*"인터뷰" + 0.005*"학교" + 0.005*"사회" + 0.005*"영어" + 0.005*"이름" + 0.004*"강사"')
    
    
    (26, '0.045*"부산" + 0.043*"농가" + 0.039*"명품" + 0.013*"부산시" + 0.012*"작물" + 0.012*"금융위" + 0.009*"투자자" + 0.008*"길드" + 0.008*"온투업" + 0.008*"농협" + 0.008*"심사" + 0.008*"유닛" + 0.007*"P2P" + 0.007*"상점" + 0.006*"요건" + 0.006*"투자연계" + 0.006*"온라인투자" + 0.005*"시계" + 0.005*"P2P금융" + 0.005*"인증서"')
    
    
    (32, '0.022*"5G" + 0.013*"강소" + 0.012*"LG유플러스" + 0.012*"소기업" + 0.010*"저축은행" + 0.009*"연구개발" + 0.009*"소부장" + 0.009*"창원" + 0.009*"고용창출" + 0.008*"경북" + 0.008*"지역" + 0.008*"대전" + 0.008*"텔레콤" + 0.007*"센터장" + 0.007*"경남" + 0.007*"기술개발" + 0.007*"혁신기업" + 0.006*"부품" + 0.006*"생산" + 0.006*"제조업"')
    
    
    (9, '0.031*"프로그램" + 0.018*"지역" + 0.018*"벤처기업" + 0.014*"선발" + 0.013*"중소벤처" + 0.013*"청년" + 0.012*"코로나" + 0.011*"사회" + 0.011*"아이디어" + 0.010*"투자유치" + 0.010*"성과" + 0.010*"지원사업" + 0.009*"기업부" + 0.009*"창업기업" + 0.008*"평가" + 0.007*"입주" + 0.007*"프로젝트" + 0.007*"멘토링" + 0.007*"경제" + 0.006*"개최"')
    
    
    (35, '0.033*"임상" + 0.033*"치료제" + 0.031*"신약" + 0.025*"바이오" + 0.013*"임상시험" + 0.011*"의약품" + 0.010*"세포" + 0.010*"약물" + 0.009*"후보물질" + 0.009*"치료" + 0.009*"약개발" + 0.008*"항체" + 0.008*"연구개발" + 0.007*"계약" + 0.007*"제약사" + 0.007*"파이프라인" + 0.007*"질환" + 0.007*"승인" + 0.006*"효능" + 0.006*"물질"')
    
    
    (19, '0.068*"대회" + 0.060*"선수" + 0.026*"시즌" + 0.023*"우승" + 0.019*"라운드" + 0.015*"스포츠" + 0.015*"리그" + 0.012*"챔피언십" + 0.012*"출전" + 0.012*"E스포츠" + 0.012*"기록" + 0.010*"갤럭시" + 0.010*"구단" + 0.010*"상금" + 0.009*"감독" + 0.009*"올림픽" + 0.009*"영입" + 0.009*"야구" + 0.009*"국가대표" + 0.008*"결승전"')
    
    


### 문서별 토픽 지정


```python
doc_topics = []
for idx, doc in tqdm(enumerate(corpus)):
    topics = ldamodel[doc]
    doc_topic = max(topics, key = itemgetter(1))
    chk = min(ldamodel[doc], key = itemgetter(1))[1]
    
    if len(topics) != 1 and doc_topic[1] == chk :
        doc_topics.append((idx+1,np.nan))
    else:
        doc_topics.append((idx+1, doc_topic[0]))

```

    17312it [00:57, 299.78it/s]



```python
# 뉴스-토픽 정보 확인

id = 14462

doc_tp = ldamodel[corpus[id-1]]
print(f'문서가 해당되는 TOPIC : { doc_tp } \n')
print(f'문서가 해당되는 TOPIC 개수 : { len(doc_tp) }')
print(f'최대값을 갖는 TOPIC : { max(doc_tp,  key = itemgetter(1)) }')
print(f'최소값을 갖는 TOPIC : { min(doc_tp,  key = itemgetter(1)) }', '\n')
print(f'본문: { morp_obj[id-1] }', '\n')
print(f'{ keywords[id-1] }')

(len(doc_tp) != 1) and (max(doc_tp,  key = itemgetter(1))[1] == min(doc_tp,  key = itemgetter(1))[1])
```

    문서가 해당되는 TOPIC : [(0, 0.025), (1, 0.025), (2, 0.025), (3, 0.025), (4, 0.025), (5, 0.025), (6, 0.025), (7, 0.025), (8, 0.025), (9, 0.025), (10, 0.025), (11, 0.025), (12, 0.025), (13, 0.025), (14, 0.025), (15, 0.025), (16, 0.025), (17, 0.025), (18, 0.025), (19, 0.025), (20, 0.025), (21, 0.025), (22, 0.025), (23, 0.025), (24, 0.025), (25, 0.025), (26, 0.025), (27, 0.025), (28, 0.025), (29, 0.025), (30, 0.025), (31, 0.025), (32, 0.025), (33, 0.025), (34, 0.025), (35, 0.025), (36, 0.025), (37, 0.025), (38, 0.025), (39, 0.025)] 
    
    문서가 해당되는 TOPIC 개수 : 40
    최대값을 갖는 TOPIC : (0, 0.025)
    최소값을 갖는 TOPIC : (0, 0.025) 
    
    본문: (14462, '') 
    
    {}





    True






### 데이터프레임 생성


```python
morp_df= pd.DataFrame(morp_obj, columns=['id', 'body'])
topic_df = pd.DataFrame(doc_topics, columns=['id', 'topic'])
```


```python
news_topic_df = pd.merge(morp_df, topic_df, how='left')
```

pd.set_option('display.max_rows', None)


```python
news_topic_df.loc[news_topic_df.topic == 0]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>body</th>
      <th>topic</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>302</th>
      <td>303</td>
      <td>LG 트윈스 홍창기, 위스타트지역아동센터에 건강기능식품 기부 [프레스나인] 조아제약...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>304</th>
      <td>305</td>
      <td>조아제약이 소외 계층 아이들을 위해 위스타트지역아동센터에 건강기능식품을 후원했다. ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1647</th>
      <td>1648</td>
      <td>온산제련소 협력사 28곳에 안전교육 지원키로 고려아연(회장 최창근)이 온산제련소 협...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1650</th>
      <td>1651</td>
      <td>고려아연(주) 온산제련소는 4일 사내 안전교육장에서 회사와 협력사가 참여하는 안전보...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1651</th>
      <td>1652</td>
      <td>▲ 고려아연㈜ 온산제련소는 4일 사내 안전교육장에서 협력사가 참여하는 안전보건경영시...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1906</th>
      <td>1907</td>
      <td>국순당이 세계 문화계 리더들이 참여하는 문화소통포럼(CCF) 2021 참석자 선물로...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1907</th>
      <td>1908</td>
      <td>[사진=국순당] [이뉴스투데이 박예진 기자] 국순당이 세계 문화계 리더가 참여하는 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1908</th>
      <td>1909</td>
      <td>국순당 자양강장 백세주 선물세트. (사진=국순당) 국순당은 세계 문화계 리더들이 참...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1909</th>
      <td>1910</td>
      <td>국순당 자양강장 백세주 선물세트 [핀포인트뉴스 이정훈 기자] 국순당은 세계 문화계 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1910</th>
      <td>1911</td>
      <td>국순당이 문화소통포럼 2021에 참석하는 세계 문화계 인사들에게 '자양강장백세주 선...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1911</th>
      <td>1912</td>
      <td>국순당은 세계 문화계 리더들이 참여하는 문화 행사인 '문화소통포럼(CCF) 202...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1912</th>
      <td>1913</td>
      <td>자양강장 백세주 선물세트. (제공: 국순당) [천지일보=황해연 기자] 국순당이 세계...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1913</th>
      <td>1914</td>
      <td>[FETV=최남주 기자] 국순당은 세계 문화계 리더들이 참여하는 문화 행사인 ‘문화...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1914</th>
      <td>1915</td>
      <td>[이데일리 전재욱 기자] 국순당은 세계 문화계 리더들이 참여하는 문화 행사인 ‘문...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2083</th>
      <td>2084</td>
      <td>김구환 대표(사진 중앙)와 챌린지에 참여한 직원들.김구환 그리드위즈 대표가 '어린...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2261</th>
      <td>2262</td>
      <td>“나눔은 함께 오래 멀리 갈 수 있는 힘입니다.” 글로벌 시장을 대상으로 인공지능(...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2411</th>
      <td>2412</td>
      <td>복권수탁사업자인 동행복권의 김세중 대표가 어린이 교통사고 예방과 안전한 교통 문화...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2412</th>
      <td>2413</td>
      <td>28일 동행복권이 김세중 동행복권 대표가 어린이 교통사고 예방과 안전한 교통 문화...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2413</th>
      <td>2414</td>
      <td>동행복권의 김세중 대표가 어린이 교통사고 예방과 안전한 교통 문화 정착을 위한 ‘...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2414</th>
      <td>2415</td>
      <td>김세중 동행복권 대표가 ‘어린이 교통안전 릴레이 챌린지’ 참여했다. 사진=동행복권 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2415</th>
      <td>2416</td>
      <td>김세중 동행복권 대표(왼쪽)와 복권판매인이 ‘어린이 교통안전 릴레이 챌린지’에 참여...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2416</th>
      <td>2417</td>
      <td>복권수탁사업자 ㈜동행복권 김세중 대표가 어린이 교통사고 예방과 안전한 교통 문화 정...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2766</th>
      <td>2767</td>
      <td>제주시소통협력센터(센터장 민복기)는 배지훈 나인투원 일레클 대표의 지목을 받아 '어...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2767</th>
      <td>2768</td>
      <td>[머니투데이 최태범 기자] 공유 전기자전거 '일레클' 운영사 나인투원의 배지훈 대...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2768</th>
      <td>2769</td>
      <td>[이코노믹리뷰=최진홍 기자] 공유 전기 자전거 일레클을 운영하는 나인투원이 11일 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>3442</th>
      <td>3443</td>
      <td>▲ 코로나19 조기 종식을 기원하는 스테이 스트롱 캠페인 동참(왼쪽부터 김용태 대전...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>3781</th>
      <td>3782</td>
      <td>청주대는 통합결제 전문기업 '다날'의 사내벤처기업인 '다날유니펀'과 '유니펀' 서비...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>3786</th>
      <td>3787</td>
      <td>[한국대학신문 조영은 기자] 청주대학교(총장 차천수)는 통합결제 전문기업 ‘다날’의...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>3789</th>
      <td>3790</td>
      <td>청주대학교는 통합결제 전문기업 다날의 사내벤처기업인 다날유니펀과 서비스 제휴 협약을...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4983</th>
      <td>4984</td>
      <td>▲ 박상원 대표(앞줄 가운데)와 동아건설산업 임직원들이 안전경영선포식에서 안전보건경...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4984</th>
      <td>4985</td>
      <td>SM그룹 건설부문 계열사 동아건설산업은 최근 본사 강당에서 안전경영 선포식을 가졌다...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4985</th>
      <td>4986</td>
      <td>동아건설산업 박상원 대표(앞줄 가운데)와 임직원들이 안전경영선포식을 갖고 파이팅을 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4988</th>
      <td>4989</td>
      <td>[로이슈 편도욱 기자]동아건설산업 박상원 대표(앞줄 가운데)와 임직원들이 안전경영선...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4989</th>
      <td>4990</td>
      <td>동아건설산업 박상원 대표(앞줄 가운데)와 임직원들이 안전경영선포식을 갖고 파이팅을 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4990</th>
      <td>4991</td>
      <td>박상원 대표(앞줄 가운데)와 임직원들이 안전경영선포식을 갖고 기념사진을 찍고 있다...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>5346</th>
      <td>5347</td>
      <td>[프라임경제] 한국온라인광고협회 협회장이자 디베이스앤 대표인 목영도 회장이 코로나1...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>5935</th>
      <td>5936</td>
      <td>백설공주 머리핀 카드뮴 기준치 10651배 초과싸인펜 등 문구에서 가소제와 방부제...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>6317</th>
      <td>6318</td>
      <td>[충청투데이 나운규 기자] 코로나19(이하 코로나) 사태에도 충청권 나눔 온도탑이 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>6318</th>
      <td>6319</td>
      <td>[예산 홍성=대전CBS 김화영 기자]충남 희망2021나눔캠페인 사랑의 온도 153...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>6319</th>
      <td>6320</td>
      <td>사랑의온도탑 153도의 뜨거운 충남 나눔 열기 내포 김관태 기자 = 충남사회복지공동...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>7618</th>
      <td>7619</td>
      <td>[사진=영영키친] [이뉴스투데이 박진선 기자] 공유주방서비스 영영키친의 조영훈 대표...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>8238</th>
      <td>8239</td>
      <td>헬로티 함수미 기자 | 스트라타시스 코리아 문종윤 지사장이 ‘어린이 교통안전 릴레이...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>8239</th>
      <td>8240</td>
      <td>문종윤 스트라타시스 코리아 지사장 [라포르시안] 3D 프린팅 솔루션 전문기업 스트라...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>8240</th>
      <td>8241</td>
      <td>[의학신문·일간보사=오인규 기자] 3D프린팅 솔루션 선도 기업 스트라타시스 코리아 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>8241</th>
      <td>8242</td>
      <td>더블에이엠 황혜영 대표 지명받아 참여... "일상 속 어린이 안전 다짐"3D프린팅...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>8242</th>
      <td>8243</td>
      <td>스트라타시스 문종윤 지사장, ‘어린이 교통안전 챌린지’ 동참 ▲ 문종윤 스트라타시스...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>10569</th>
      <td>10570</td>
      <td>[머니투데이 김건우 기자] 어린이 교통안전 릴레이 챌린지에 동참한 글로벌텍스프리 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>10570</th>
      <td>10571</td>
      <td>[머니투데이 김건우 기자] 브이원텍 김선중 대표가 어린이 교통안전 릴레이 챌린지'...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>10571</th>
      <td>10572</td>
      <td>김재영 제테마 대표가 어린이 교통안전 릴데이 챌린지에 동참하는 모습 [제테마 제공]...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>11386</th>
      <td>11387</td>
      <td>걸음기부 플랫폼을 운영하는 IT 사회적기업 빅워크가 커뮤니티형 챌린지 서비스를 출시...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>11390</th>
      <td>11391</td>
      <td>-걷는 것만으로도 선한 영향력 실현사진: 빅워크 제공빅워크는 ESG(환경·사회·지...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>11391</th>
      <td>11392</td>
      <td>[세계비즈=박보라 기자] IT 사회적기업 빅워크가 커뮤니티형 챌린지 서비스를 출시했...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>11392</th>
      <td>11393</td>
      <td>모바일 걸음 기부 서비스 ‘빅워크’가 사회 전반에 선한 영향력을 전하기 위해 ESG...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>11393</th>
      <td>11394</td>
      <td>[비지니스코리아=윤영실 기자] 빅워크가 선한 영향력 캠페인을 진행한다고 27일 밝혔...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>11394</th>
      <td>11395</td>
      <td>빅워크가 선한 영향력 캠페인을 진행한다고 27일 밝혔다. 빅워크 제공 빅워크 앱은 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>11895</th>
      <td>11896</td>
      <td>수상작 '개나리 울타리'제5회 문학청춘작품상, 시인 김환식[이데일리 장병호 기자]...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>11896</th>
      <td>11897</td>
      <td>[서울=뉴시스]시인 김기택 (사진 = 문학청춘 제공) 2021.8.30. phot...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>12393</th>
      <td>12394</td>
      <td>코맥스벤처러스의 육성 기업 ‘세차왕’의 박정률 대표이사가 ‘어린이 교통안전 릴레이 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>12394</th>
      <td>12395</td>
      <td>안상민 기자 작성 : 2021년 07월 06일(화) 12:48 게시 : 2021년 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>13153</th>
      <td>13154</td>
      <td>▲ 5월 7일(금) 한국교통안전공단 조경수 교통안전본부장(오른쪽)과 주식회사 슈어소...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>13300</th>
      <td>13301</td>
      <td>㈜스마트올리브, 굿피플과 함께 지역사회 취약계층 위한 사회공헌활동 모색해 나갈 예정...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>13936</th>
      <td>13937</td>
      <td>이명재 롯데손해보험 대표이사가 ‘어린이 교통안전 릴레이 챌린지’에 참여했다. [사진...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>13938</th>
      <td>13939</td>
      <td>최동수 기자 승인 2021.07.07 17:20 의견 0 X 이명재 롯데손해보험 대...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>13939</th>
      <td>13940</td>
      <td>▲ 이명재 롯데손해보험 대표이사가 '어린이 교통안전 릴레이 챌린지'에 참여해 피켓을...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>13940</th>
      <td>13941</td>
      <td>신한라이프가 론칭한 TV 및 사회관계망서비스(SNS) 광고에 등장하는 가상캐릭터 '...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>13941</th>
      <td>13942</td>
      <td>[뉴스토마토 권유승 기자] 롯데손해보험은 이명재 대표이사가 '어린이 교통안전 릴레이...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>13942</th>
      <td>13943</td>
      <td>ⓒ롯데손해보험 롯데손해보험은 이명재 대표이사가 '어린이 교통안전 릴레이 챌린지'에 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>13943</th>
      <td>13944</td>
      <td>이명재 대표이사 챌린지 사진 [핀포인트뉴스 박채원 기자]7일 롯데손해보험은 이명재 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>13944</th>
      <td>13945</td>
      <td>다음 참여자로 안치홍 밀리만코리아 대표·심찬구 스포티즌 대표 지목 (사진=롯데손해보...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14035</th>
      <td>14036</td>
      <td>이상훈 시스원 대표(오른쪽 두번째)가 장애인거주시설 '애덕의 집'으로부터 '선한이...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14036</th>
      <td>14037</td>
      <td>맨 왼쪽 김은순 대데레사 부장수녀, 가운데 김경자 안나마리아 원장수녀, 이상훈 대...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14226</th>
      <td>14227</td>
      <td>불필요한 플리스틱 사용 줄이기 대한제강이 18일 ‘고고 챌린지(Go!Go!Chall...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14293</th>
      <td>14294</td>
      <td>(시사오늘, 시사ON, 시사온=방글 기자) 박현종 알피니언메디칼시스템 대표가 어린이...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14462</th>
      <td>14463</td>
      <td>사랑의열매 사회복지공동모금회(회장 조흥식)는 80대 노신사가 10억 원을 기부했다...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14463</th>
      <td>14464</td>
      <td>신원식 태양연마 회장, 한국형 기부자맞춤기금 10호 가입자[경향신문] 아너 소사이...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14464</th>
      <td>14465</td>
      <td>1961년 창업 '연매출 1000억'한국형 기부자맞춤기금 10호"받았던 도움 나눌...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14465</th>
      <td>14466</td>
      <td>신원식 태양연마 회장"저소득층에 잘 전해졌으면" 평생 연마지 제조로 재산을 일군 ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14466</th>
      <td>14467</td>
      <td>사랑의열매에 10억원을 기부한 신원식(83) 태양연마 회장. /사랑의열매 제공 80...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14467</th>
      <td>14468</td>
      <td>신원식 (주)태양연마 회장. ⓒ천지일보 2021.5.26 [천지일보=홍보영 기자] ...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14468</th>
      <td>14469</td>
      <td>“도움 주자는 다짐 실천하기 위해 기부”사랑의열매 ‘기부자 맞춤 기금’ 10호 가...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14929</th>
      <td>14930</td>
      <td>[공감신문] 권오선 기자 = 아샤그룹(대표 이은영)에서 남양주시 복지관 서부희망케어...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>14930</th>
      <td>14931</td>
      <td>아샤그룹(대표 이은영)에서 남양주시 복지관 동부희망케어센터에 자사 브랜드 ‘셀로몬...</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>15489</th>
      <td>15490</td>
      <td>'어린이 교통안전 릴레이 캠페인'에 참여한 정상억 파라투스인베스트먼트 대표이사[출...</td>
      <td>0.0</td>
    </tr>
  </tbody>
</table>
</div>


# 토픽 개수 찾기
- 02.FindTopicNum_code.ipynb
