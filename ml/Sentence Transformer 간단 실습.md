# Sentence Transformer 간단 실습

텍스트를 임베딩하고 텍스트 간의 유사도를 확인하기 위해 Sentence Transformer를 사용하게 되었다.  

```python
pip install sentence-transformers
```

```python
model = SentenceTransformer('all-MiniLM-L6-v2')

sentences = ['This framework generates embeddings for each input sentence', 'Sentences are passed as a list of string.']

embeddings = model.encode(sentences)

for sentence, embedding in zip(sentences, embeddings):
    print("sentence: ", sentence)
    print("embedding: ", embedding)
    print("")
```

단순히 모델을 지정핳고 encode를 하면 임베딩된다.  

```python
from sentence_transformers import SentenceTransformer, util

emb1 = model.encode("i like shopping through internet")
emb2 = model.encode("line shopping is one of the e-commerce platform")

cos_sim = util.cos_sim(emb1, emb2)

print("Cosine-Similarity: ", cos_sim)
```

그리고 util의 cos_sim을 통해 코사인 유사도를 계산할 수 있다.  

```python
sentences = ['A man is eating food.',
          'A man is eating a piece of bread.',
          'The girl is carrying a baby.',
          'A man is riding a horse.',
          'A woman is playing violin.',
          'Two men pushed carts through the woods.',
          'A man is riding a white horse on an enclosed ground.',
          'A monkey is playing drums.',
          'Someone in a gorilla costume is playing a set of drums.'
          ]

#Encode all sentences
embeddings = model.encode(sentences)

#Compute cosine similarity between all pairs
cos_sim = util.cos_sim(embeddings, embeddings)

#Add all pairs to a list with their cosine similarity score
all_sentence_combinations = []
for i in range(len(cos_sim)-1):
    for j in range(i+1, len(cos_sim)):
        all_sentence_combinations.append([cos_sim[i][j], i, j])

#Sort list by the highest cosine similarity score
all_sentence_combinations = sorted(all_sentence_combinations, key=lambda x: x[0], reverse=True)

print("Top-5 most similar pairs:")
for score, i, j in all_sentence_combinations[0:5]:
    print("{} \t {} \t {:.4f}".format(sentences[i], sentences[j], cos_sim[i][j]))
```

여러 문장간의 유사도를 구해서 유사도 내림차순으로 5개의 문장들을 출력

```python
from sentence_transformers import SentenceTransformer, util
model = SentenceTransformer('multi-qa-MiniLM-L6-cos-v1')

query_embedding = model.encode('How big is London')
passage_embedding = model.encode(['London has 9,787,426 inhabitants at the 2011 census',
                                  'London is known for its finacial district'])

print("Similarity:", util.dot_score(query_embedding, passage_embedding))
```

공홈 docs를 보는게 빠른 것 같다...

---

### REFERENCE

https://www.sbert.net/
