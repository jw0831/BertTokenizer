# BertStudy - Tokenization
Google에서 제공하는 NLP 모델인 BERT를 통해 한글어 자연어 처리를 시도하는 과정에서 더 나은 방식을 찾고 정리하고 있습니다.
해당 레파짓토리에서는 BERT 학습에 가장 근본이 되는 토크나이저 부분에 대해 정리하고 개선하는 부분입니다.



## 토크나이저 개선 사항
BERT를 통해 한글 의도 분류 모델을 만들기 위해서 가장 처음 봉착했던 문제가 한글 사전 및 토크나이저 오류 문제입니다.
개발자도 아니고 파이썬도 익숙치 않던 때라 워드피스 토크나이징이란 개념도 전혀 몰랐고 한줄 한줄 프린트 해보며 한글 데이터도 돌아가도록 했었죠.   
차츰 워드피스 토크나이징의 개념이 이해되면서 더 나은 사전을 빠르게 만드는 방법에 대해 생각도 해보게 되었고 한글 데이터에 맞도록 토크나이저도 변경하게 되었습니다.

**Tokenization_kor.py** 파일 참조

|개선사항|변경 부분|변경 이유|예시|
|:-----|:---:|:---:|:---:|
|한글 처리|token = self._run_strip_accents(token) 주석 처리|한글 NFD 시 글자 깨짐 현상|ㄱ ㅣㅁ ㅇ ㅣㄹ ㅎㅜㄴ > 김일훈|
|의미, 빈공간 구분| Full tokenizer 에서 token += "_" 추가 |의미 구분 및 스페이스 공간 구분|인가 를_ 받기_ 위해_ 필요한_ 준비물 은_ 무엇 인가_|
|구분자 기준 나누기 비활성화|_is_punctuation 함수 비활성화|기호가 문장에 붙어있음에도 불구하고 띄우는 현상 제거|나 는_ 밥 을_ 먹었다_. > 나 는_ 밥 을_ 먹었다 .|
|종결 토큰 처리|FullTokenizer 수정|문장의 끝에 무조건 "\_"가 붙었던 문제 처리|나 는_ 밥 을_ 먹었다 ._  > 나 는_ 밥 을_ 먹었다 .|


## 한글 사전 만들기
**get_vocab.ipynb** 파일 참조

### 1. Google sentencepiece 활용 사전 만들기

해당 내용은 [구글 sentencepiece github](https://github.com/google/sentencepiece)을 참조하면 이해하기 쉽습니다.   
먼저 sentencepiece 라이브러리를 불러옵니다.
```
  import sentencepiece as spm
```
하이퍼파라미터를 설정합니다.
```
train ="""--input=./data/ratings_train.txt \
    --model_prefix=sentpiece \
    --vocab_size=30000 \
    --model_type=bpe --character_coverage=0.9995"""
```
학습을 하면 모델, 사전 파일이 생깁니다.
```
spm.SentencePieceTrainer.Train(train)
```

학습된 모델을 로드합니다.
```
sp = spm.SentencePieceProcessor()
sp.Load('sentpiece.model')
```
Sentencepiece 로 학습한 모델, 사전으로 토크나이징한 결과는 다음과 같습니다.
```
>>> sp.EncodeAsPieces('이 영화는 정말 기분나쁘고 소름돋아서 다시는 보기 싫었어요.')
...
['▁이', '▁영화는', '▁정말', '▁기분나쁘', '고', '▁소름돋', '아서', '▁다시는', '▁보기', '▁싫', '었어요', '.']
```
### 2. Konlpy 활용하여 명사 추출 및 반복 조사 제거
구글 Sentencepiece으로 만들어진 사전을 확인해보면 '영화는'과 같이 **'명사+조사'** 의 형태가 많은 것을 알 수 있습니다.    
이런 문제를 해결하기 위해서 저는 오픈소스 형태소 분석기로 추출된 사전 단어를 재분해하는 생각을 했습니다. 

```
>>> from konlpy.tag import Okt
>>> okt = Okt()
>>> okt.pos("영화는")
...
[('영화', 'Noun'), ('는', 'Josa')]
```

이런식으로 사전 단어에서 명사를 추출하고 해당 단어에서 명사를 제거한 부분만 추출하였습니다. 그리고 중복제거를 통해 사전을 만들게 됩니다.
```
nouns = []
removal_nouns = []
for i in vocabs["word"]:
    extract_nouns = okt.nouns(i)
    if len(extract_nouns) == 1:
        nouns.append("".join(extract_nouns))           
        removal = i.replace("".join(extract_nouns),"") 
        removal_nouns.append(removal)
        
    else:
        nouns.append("")
        removal_nouns.append(i)  
```

### 3. WordPiece Tokenizing 개선하기
word piece 방식의 토크나이징의 경우 동음이의어(조사와 명사 동음이의어)를 구분하지 못한다는 단점이 있습니다.

이러한 점을 개선하기 위해서 '\_'를 token에 붙입니다.
```
u_final_vocab=[]
for i in final_vocab:
    u = f'{i}_'
    u_final_vocab.append(u)
    u_final_vocab.append(i)  
```

모든 사전 단어에 '\_'를 붙였기 때문에 불필요하게 사전 단어가 늘어있는 상태라 실제 사용된 토큰만 다시 추출합니다.

우선 **tokenizer_kor.py** 를 사용하기 위해 구글 임베딩용 토큰을 넣어주고 txt 파일로 저장합니다.
```
bert_specific=["[UNK]","[CLS]","[SEP]","[MASK]"]
for i in bert_specific:
    u_final_vocab.append(i)
    
fin_vocab = pd.DataFrame(data = u_final_vocab, columns=None, index =None)
fin_vocab.to_csv("./test_vocab.txt", header=None, index=None)
```

방금 만든 사전을 기반으로 토크나이저를 로드합니다.
```
>>> import tokenization_kor as tokenization
>>> tokenizer = tokenization.FullTokenizer("test_vocab.txt")
```

전체 말뭉치를 해당 토크나이저로 토크나이징하고 실제 쓰인 토큰을 중복없이 리스트에 넣어주면 최종 사전이 완성됩니다.
```
data = pd.read_csv("./data/ratings_train.txt",sep="\t")

use_vocab = []
for sentence in tqdm(data["document"]):
    try:
        token = tokenizer.tokenize(sentence)
        for t in token:
            if t not in use_vocab:
                use_vocab.append(t)
    except:
        pass
```

그냥 구글 Sentencepiece를 사용하여 학습한 사전과 추가 작업을 한 최종 사전을 통한 토크나이징 결과의 차이입니다.
```
>>> sp.EncodeAsPieces('이 영화는 정말 기분나쁘고 소름돋아서 다시는 보기 싫었어요.')
....
['▁이', '▁영화는', '▁정말', '▁기분나쁘', '고', '▁소름돋', '아서', '▁다시는', '▁보기', '▁싫', '었어요', '.']

>>> tokenizer.tokenize('이 영화는 정말 기분나쁘고 소름돋아서 다시는 보기 싫었어요.')
....
['이_', '영화', '는_', '정말_', '기분', '나쁘', '고_', '소름', '돋', '아서_', '다시는_', '보기_', '싫', '었어', '요', '.']
```

이러한 과정을 모두 거치고 나서 실제로 사전 파일을 엑셀로 열어 다시 확인하는 편입니다.   
전처리 과정이 부족한 부분을 찾기도 하고 의미없는 숫자나 영어같은 경우는 제거하고 OOV 측면에서도 고려하여 임베딩이 되도록 영어,숫자는 모든 알파벳과 숫자로 넣어주곤 합니다.
