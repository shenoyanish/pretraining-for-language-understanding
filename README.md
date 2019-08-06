# Pre-training For Language-Understanding
Generalized Pre-training for Language Understanding 

## Overview
### Language Modeling (LM)
A language model captures **the distribution over all possible sentences**.
- Input : a sentence
- Output : the probability of the input sentence

It is _unsupervised learning_. In this repo, we turn this into a _sequence of supervised learning_.

### Autoregressive LM
The Autoregressive language model looks at the previous token and predicts the next token.
ELMo, GPT RNNLM are typically the case.

<br>
<p align="center">
<img width="500" src="https://storage.googleapis.com/deepmind-live-cms/documents/BlogPost-Fig2-Anim-160908-r01.gif" align="middle">
</p>
<br>

Because Autoregressive LM should be forward or backward, only one-way(uni-directional) context information can be used.
Therefore, it's difficult to understand the context in both directions(bi-directional).


## Build Corpus (Wikipedia)
Wikipedia regularly distributes the entire document. You can download Korean Wikipedia dump [here](https://dumps.wikimedia.org/kowiki/) (and English Wikipedia dump [here](https://dumps.wikimedia.org/enwiki/)).
Wikipedia recommends using `pages-articles.xml.bz2`, which includes only the latest version of the entire document, and is approximately 600 MB compressed (for English, `pages-articles-multistream.xml.bz2`).

You can use `wikipedia_ko.sh` script to download the dump on the latest Korean Wikipedia document. For English, use `wikipedia_en.sh`

example:
```
$ cd build_corpus
$ chmod 777 wikipedia_ko.sh
$ ./wikipedia_ko.sh
```

The downloaded dump using above shell script is in XML format, and we need to parse XML to text file. The Python script `WikiExtractor.py` in [attardi/wikiextractor](https://github.com/attardi/wikiextractor) repo, extracts and cleans text from the dump.

example:
```
$ git clone https://github.com/attardi/wikiextractor
$ python wikiextractor/WikiExtractor.py kowiki-latest-pages-articles.xml

$ head -n 4 text/AA/wiki_02
>> <doc id="577" url="https://ko.wikipedia.org/wiki?curid=577" title="천문학">
>> 천문학
>>
>> 천문학(天文學, )은 별이나 행성, 혜성, 은하와 같은 천체와, 지구 대기의 ..
>> ...
>> </doc>
```

The extracted text is saved as text file of a certain size. To combine these, use `build_corpus.py`. The output `corpus.txt` contains _4,277,241 sentences, 55,568,030 words_.

example:
```
$ python build_corpus.py > corpus.txt
$ wc corpus.txt 
4277241  55568030 596460787 corpus.txt
```

Now, you need to split the corpus to train-set and test-set.

```
$ cat corpus.txt | shuf > corpus.shuf.txt
$ head -n 855448 corpus.shuf.txt > corpus.test.txt
$ tail -n 3421793 corpus.shuf.txt > corpus.train.txt
$ wc -l corpus.train.txt corpus.test.txt
  3421793 corpus.train.txt
   855448 corpus.test.txt
  4277241 합계
```
 
## Preprocessing

### Build Vocab
Our corpus `corpus.shuf.txt`(or `corpus.txt`) has _55,568,030_ words, and _608,221_ unique words. If the minimum frequency needed to include a token in the vocabulary is set to 3, the vocabulary contains **_297,773_** unique words.

Here we use the train corpus `corpus.train.txt` to build vocabulary.
The vocabulary built by train corpus contains **_557,627_** unique words, and **_271,503_** unique words that appear at least three times.

example:
```
$ python build_vocab.py --corpus build_corpus/corpus.train.txt --vocab vocab.train.pkl --min_freq 3 --lower
Namespace(bos_token='<bos>', corpus='build_corpus/corpus.train.txt', eos_token='<eos>', is_tokenized=False, lower=True, min_freq=3, pad_token='<pad>', tokenizer='mecab', unk_token='<unk>', vocab='vocab.train.pkl')
Vocabulary size:  271503
Vocabulary saved to vocab.train.pkl
```

Since the vocab file is too large(~1.3GB) to upload on Github, I uploaded it to Google Drive.
you can download vocab file `vocab.train.pkl` in [here](https://drive.google.com/file/d/195kdXPQtiG0eqppH-L2VKoHAcgqCSR1l/view?usp=sharing).

## Training

```
$ python lm_trainer.py -h
usage: lm_trainer.py [-h] --train_corpus TRAIN_CORPUS --test_corpus
                     TEST_CORPUS --vocab VOCAB --model_type MODEL_TYPE
                     [--is_tokenized] [--tokenizer TOKENIZER]
                     [--max_seq_len MAX_SEQ_LEN] [--multi_gpu_training]
                     [--cuda CUDA] [--epochs EPOCHS] [--batch_size BATCH_SIZE]
                     [--shuffle SHUFFLE] [--embedding_size EMBEDDING_SIZE]
                     [--hidden_size HIDDEN_SIZE] [--n_layers N_LAYERS]
                     [--dropout_p DROPOUT_P]
                     [--is_bidirectional IS_BIDIRECTIONAL]

optional arguments:
  -h, --help            show this help message and exit
  --train_corpus TRAIN_CORPUS
  --test_corpus TEST_CORPUS
  --vocab VOCAB
  --model_type MODEL_TYPE
                        Model type selected in the list: LSTM
  --is_tokenized        Whether the corpus is already tokenized
  --tokenizer TOKENIZER
                        Tokenizer used for input corpus tokenization
  --max_seq_len MAX_SEQ_LEN
                        The maximum total input sequence length after
                        tokenization
  --multi_gpu_training  Whether to training with multiple GPU
  --cuda CUDA           Whether CUDA is currently available
  --epochs EPOCHS       Total number of training epochs to perform
  --batch_size BATCH_SIZE
                        Batch size for training
  --shuffle SHUFFLE     Whether to reshuffle at every epoch
  --embedding_size EMBEDDING_SIZE
                        Word embedding vector dimension
  --hidden_size HIDDEN_SIZE
                        Hidden size of LSTM
  --n_layers N_LAYERS   Number of layers in LSTM
  --dropout_p DROPOUT_P
                        Dropout rate used for dropout layer in LSTM
  --is_bidirectional IS_BIDIRECTIONAL
                        Whether to use bidirectional LSTM
```

### Training with multiple GPU

Below is the example command for training with multiple GPU. You can select your own parameter values via argument inputs. 

example:
```
$ python lm_trainer.py --train_corpus build_corpus/corpus.train.txt --test_corpus build_corpus/corpus.test.txt --vocab vocab.train.pkl --model_type LSTM --multi_gpu_training 
Namespace(batch_size=192, cuda=True, dropout_p=0.2, embedding_size=256, epochs=10, hidden_size=512, is_bidirectional=True, is_tokenized=False, max_seq_len=32, model_type='LSTM', multi_gpu_training=True, n_layers=3, shuffle=True, test_corpus='build_corpus/corpus.test.txt', tokenizer='mecab', train_corpus='build_corpus/corpus.train.txt', vocab='vocab.train.pkl')
=========MODEL=========
 DataParallelModel(
  (module): LSTMLM(
    (embedding): Embedding(271503, 256)
    (lstm): LSTM(256, 512, num_layers=3, batch_first=True, dropout=0.2, bidirectional=True)
    (fc): Linear(in_features=1024, out_features=512, bias=True)
    (fc2): Linear(in_features=512, out_features=271503, bias=True)
    (softmax): LogSoftmax()
  )
)
```

## Evaluation

The models were trained with 4 * NVIDIA Tesla V100, and the number of epochs was 10.

### Extrinsic evaluation

#### Perplexity

A language model captures the distribution over all sentences. So, the best language model is one that the best predicts an unseen sentences. And now, the perplexity is the metric that we're going to be using.

`Perplexity`: **_inverse probability of the given sentence, normalized by the number of words._** The reason why for using inverse probability is a way of normalizing for the sentence length.

<p align="center">
<img src="https://latex.codecogs.com/gif.latex?\dpi{100}&space;\fn_jvn&space;PP(W)&space;=&space;P(w_{1},&space;w_{2}...w_{n})^{-\frac{1}{n}}&space;=\sqrt[n]{\frac{1}{P(w_{1}w_{2}...w_{N})}}" title="PP(W) = P(w_{1}, w_{2}...w_{n})^{-\frac{1}{n}} =\sqrt[n]{\frac{1}{P(w_{1}w_{2}...w_{N})}}" />
</p>

<p align="center">
<img src="https://latex.codecogs.com/gif.latex?\dpi{100}&space;\fn_jvn&space;Chain\;&space;rule:\;&space;PP(W)&space;=&space;\sqrt[n]{\prod_{i=1}^{N}\frac{1}{P(w_{i}|w_{1}...w_{i-1})}}" title="Chain\; rule:\; PP(W) = \sqrt[n]{\prod_{i=1}^{N}\frac{1}{P(w_{i}|w_{1}...w_{i-1})}}" />
</p>

As you can see from the above equation, the minimizing perplexity is the same as maximizing probability.


### Intrinsic evaluation

|Input sentence|Unidirectional-LSTM|
|------|------|
|||


## Reference
- [attardi/wikiextractor] [WikiExtractor](https://github.com/attardi/wikiextractor)
- [zhanghang1989/PyTorch-Encoding] [PyTorch-Encoding](https://github.com/zhanghang1989/PyTorch-Encoding)
, [Issue: How to use the DataParallelCriterion, DataParallelModel](https://github.com/zhanghang1989/PyTorch-Encoding/issues/54)
- [Google DeepMind] [WaveNet: A Generative Model for Raw Audio](https://deepmind.com/blog/wavenet-generative-model-raw-audio/)
- [Dan Jurafsky] [CS 124: From Languages to Information at Stanford](https://web.stanford.edu/class/cs124/lec/languagemodeling2019.pdf)
