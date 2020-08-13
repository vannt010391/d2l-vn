<!-- ===================== Bắt đầu dịch Phần 1 ==================== -->
<!-- ========================================= REVISE PHẦN 1 - BẮT ĐẦU =================================== -->

<!--
# The Dataset for Pretraining Word Embedding
-->

# *dịch đoạn phía trên*
:label:`sec_word2vec_data`


<!--
In this section, we will introduce how to preprocess a dataset with negative sampling :numref:`sec_approx_train` and load into minibatches forword2vec training.
The dataset we use is [Penn Tree Bank (PTB)](https://catalog.ldc.upenn.edu/LDC99T42), which is a small but commonly-used corpus.
It takes samples from Wall Street Journal articles and includes training sets, validation sets, and test sets.
-->

*dịch đoạn phía trên*


<!--
First, import the packages and modules required for the experiment.
-->

*dịch đoạn phía trên*


```{.python .input  n=1}
from d2l import mxnet as d2l
import math
from mxnet import gluon, np
import os
import random
```



<!--
## Reading and Preprocessing the Dataset
-->

## *dịch đoạn phía trên*


<!--
This dataset has already been preprocessed.
Each line of the dataset acts as a sentence.
All the words in a sentence are separated by spaces.
In the word embedding task, each word is a token.
-->

*dịch đoạn phía trên*


```{.python .input  n=2}
#@save
d2l.DATA_HUB['ptb'] = (d2l.DATA_URL + 'ptb.zip',
                       '319d85e578af0cdc590547f26231e4e31cdf1e42')

#@save
def read_ptb():
    data_dir = d2l.download_extract('ptb')
    with open(os.path.join(data_dir, 'ptb.train.txt')) as f:
        raw_text = f.read()
    return [line.split() for line in raw_text.split('\n')]

sentences = read_ptb()
f'# sentences: {len(sentences)}'
```


<!--
Next we build a vocabulary with words appeared not greater than 10 times mapped into a "&lt;unk&gt;" token.
Note that the preprocessed PTB data also contains "&lt;unk&gt;" tokens presenting rare words.
-->

*dịch đoạn phía trên*


```{.python .input  n=3}
vocab = d2l.Vocab(sentences, min_freq=10)
f'vocab size: {len(vocab)}' 
```


<!--
## Subsampling
-->

## *dịch đoạn phía trên*


<!--
In text data, there are generally some words that appear at high frequencies, such "the", "a", and "in" in English.
Generally speaking, in a context window, it is better to train the word embedding model when a word (such as "chip") and 
a lower-frequency word (such as "microprocessor") appear at the same time, rather than when a word appears with a higher-frequency word (such as "the").
Therefore, when training the word embedding model, we can perform subsampling[2] on the words.
Specifically, each indexed word $w_i$ in the dataset will drop out at a certain probability.
The dropout probability is given as:
-->

*dịch đoạn phía trên*


$$ P(w_i) = \max\left(1 - \sqrt{\frac{t}{f(w_i)}}, 0\right),$$


<!--
Here, $f(w_i)$ is the ratio of the instances of word $w_i$ to the total number of words in the dataset, 
and the constant $t$ is a hyperparameter (set to $10^{-4}$ in this experiment).
As we can see, it is only possible to drop out the word $w_i$ in subsampling when $f(w_i) > t$.
The higher the word's frequency, the higher its dropout probability.
-->

*dịch đoạn phía trên*


```{.python .input  n=4}
#@save
def subsampling(sentences, vocab):
    # Map low frequency words into <unk>
    sentences = [[vocab.idx_to_token[vocab[tk]] for tk in line]
                 for line in sentences]
    # Count the frequency for each word
    counter = d2l.count_corpus(sentences)
    num_tokens = sum(counter.values())

    # Return True if to keep this token during subsampling
    def keep(token):
        return(random.uniform(0, 1) <
               math.sqrt(1e-4 / counter[token] * num_tokens))

    # Now do the subsampling
    return [[tk for tk in line if keep(tk)] for line in sentences]

subsampled = subsampling(sentences, vocab)
```

<!-- ===================== Kết thúc dịch Phần 1 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 2 ===================== -->

<!--
Compare the sequence lengths before and after sampling, we can see subsampling significantly reduced the sequence length.
-->

*dịch đoạn phía trên*


```{.python .input  n=5}
d2l.set_figsize()
d2l.plt.hist([[len(line) for line in sentences],
              [len(line) for line in subsampled]])
d2l.plt.xlabel('# tokens per sentence')
d2l.plt.ylabel('count')
d2l.plt.legend(['origin', 'subsampled']);
```


<!--
For individual tokens, the sampling rate of the high-frequency word "the" is less than 1/20.
-->

*dịch đoạn phía trên*


```{.python .input  n=6}
def compare_counts(token):
    return (f'# of "{token}": '
            f'before={sum([line.count(token) for line in sentences])}, '
            f'after={sum([line.count(token) for line in subsampled])}')

compare_counts('the')
```


<!--
But the low-frequency word "join" is completely preserved.
-->

*dịch đoạn phía trên*


```{.python .input  n=7}
compare_counts('join')
```


<!--
Last, we map each token into an index to construct the corpus.
-->

*dịch đoạn phía trên*


```{.python .input  n=8}
corpus = [vocab[line] for line in subsampled]
corpus[0:3]
```


<!--
## Loading the Dataset
-->

## *dịch đoạn phía trên*


<!--
Next we read the corpus with token indicies into data batches for training.
-->

*dịch đoạn phía trên*


<!-- ===================== Kết thúc dịch Phần 2 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 3 ===================== -->


<!--
### Extracting Central Target Words and Context Words
-->

### *dịch đoạn phía trên*


<!--
We use words with a distance from the central target word not exceeding the context window size as the context words of the given center target word.
The following definition function extracts all the central target words and their context words.
It uniformly and randomly samples an integer to be used as the context window size between integer 1 and the `max_window_size` (maximum context window).
-->

*dịch đoạn phía trên*


```{.python .input  n=9}
#@save
def get_centers_and_contexts(corpus, max_window_size):
    centers, contexts = [], []
    for line in corpus:
        # Each sentence needs at least 2 words to form a "central target word
        # - context word" pair
        if len(line) < 2:
            continue
        centers += line
        for i in range(len(line)):  # Context window centered at i
            window_size = random.randint(1, max_window_size)
            indices = list(range(max(0, i - window_size),
                                 min(len(line), i + 1 + window_size)))
            # Exclude the central target word from the context words
            indices.remove(i)
            contexts.append([line[idx] for idx in indices])
    return centers, contexts
```


<!--
Next, we create an artificial dataset containing two sentences of 7 and 3 words, respectively.
Assume the maximum context window is 2 and print all the central target words and their context words.
-->

*dịch đoạn phía trên*


```{.python .input  n=10}
tiny_dataset = [list(range(7)), list(range(7, 10))]
print('dataset', tiny_dataset)
for center, context in zip(*get_centers_and_contexts(tiny_dataset, 2)):
    print('center', center, 'has contexts', context)
```


<!--
We set the maximum context window size to 5.
The following extracts all the central target words and their context words in the dataset.
-->

*dịch đoạn phía trên*


```{.python .input  n=11}
all_centers, all_contexts = get_centers_and_contexts(corpus, 5)
f'# center-context pairs: {len(all_centers)}' 
```


### Negative Sampling


<!--
We use negative sampling for approximate training.
For a central and context word pair, we randomly sample $K$ noise words ($K=5$ in the experiment).
According to the suggestion in the Word2vec paper, the noise word sampling probability $P(w)$ is the ratio of 
the word frequency of $w$ to the total word frequency raised to the power of 0.75 [2].
-->

*dịch đoạn phía trên*


<!--
We first define a class to draw a candidate according to the sampling weights.
It caches a 10000 size random number bank instead of calling `random.choices` every time.
-->

*dịch đoạn phía trên*


```{.python .input  n=12}
#@save
class RandomGenerator:
    """Draw a random int in [0, n] according to n sampling weights."""
    def __init__(self, sampling_weights):
        self.population = list(range(len(sampling_weights)))
        self.sampling_weights = sampling_weights
        self.candidates = []
        self.i = 0

    def draw(self):
        if self.i == len(self.candidates):
            self.candidates = random.choices(
                self.population, self.sampling_weights, k=10000)
            self.i = 0
        self.i += 1
        return self.candidates[self.i-1]

generator = RandomGenerator([2, 3, 4])
[generator.draw() for _ in range(10)]
```



```{.python .input  n=13}
#@save
def get_negatives(all_contexts, corpus, K):
    counter = d2l.count_corpus(corpus)
    sampling_weights = [counter[i]**0.75 for i in range(len(counter))]
    all_negatives, generator = [], RandomGenerator(sampling_weights)
    for contexts in all_contexts:
        negatives = []
        while len(negatives) < len(contexts) * K:
            neg = generator.draw()
            # Noise words cannot be context words
            if neg not in contexts:
                negatives.append(neg)
        all_negatives.append(negatives)
    return all_negatives

all_negatives = get_negatives(all_contexts, corpus, 5)
```

<!-- ===================== Kết thúc dịch Phần 3 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 4 ===================== -->

<!-- ========================================= REVISE PHẦN 1 - KẾT THÚC ===================================-->

<!-- ========================================= REVISE PHẦN 2 - BẮT ĐẦU ===================================-->

<!--
### Reading into Batches
-->

### *dịch đoạn phía trên*


<!--
We extract all central target words `all_centers`, and the context words `all_contexts` and noise words `all_negatives` of each central target word from the dataset.
We will read them in random minibatches.
-->

*dịch đoạn phía trên*


<!--
In a minibatch of data, the $i^\mathrm{th}$ example includes a central word and its corresponding $n_i$ context words and $m_i$ noise words.
Since the context window size of each example may be different, the sum of context words and noise words, $n_i+m_i$, will be different.
When constructing a minibatch, we concatenate the context words and noise words of each example, 
and add 0s for padding until the length of the concatenations are the same, that is, the length of all concatenations is $\max_i n_i+m_i$(`max_len`).
In order to avoid the effect of padding on the loss function calculation, we construct the mask variable `masks`, 
each element of which corresponds to an element in the concatenation of context and noise words, `contexts_negatives`.
When an element in the variable `contexts_negatives` is a padding, the element in the mask variable `masks` at the same position will be 0.
Otherwise, it takes the value 1.
In order to distinguish between positive and negative examples, we also need to distinguish the context words from the noise words in the `contexts_negatives` variable.
Based on the construction of the mask variable, we only need to create a label variable `labels` with the same shape 
as the `contexts_negatives` variable and set the elements corresponding to context words (positive examples) to 1, and the rest to 0.
-->

*dịch đoạn phía trên*


<!--
Next, we will implement the minibatch reading function `batchify`.
Its minibatch input `data` is a list whose length is the batch size, each element of which contains central target words `center`, context words `context`, and noise words `negative`.
The minibatch data returned by this function conforms to the format we need, for example, it includes the mask variable.
-->

*dịch đoạn phía trên*


```{.python .input  n=14}
#@save
def batchify(data):
    max_len = max(len(c) + len(n) for _, c, n in data)
    centers, contexts_negatives, masks, labels = [], [], [], []
    for center, context, negative in data:
        cur_len = len(context) + len(negative)
        centers += [center]
        contexts_negatives += [context + negative + [0] * (max_len - cur_len)]
        masks += [[1] * cur_len + [0] * (max_len - cur_len)]
        labels += [[1] * len(context) + [0] * (max_len - len(context))]
    return (np.array(centers).reshape(-1, 1), np.array(contexts_negatives),
            np.array(masks), np.array(labels))
```


<!--
Construct two simple examples:
-->

*dịch đoạn phía trên*


```{.python .input  n=15}
x_1 = (1, [2, 2], [3, 3, 3, 3])
x_2 = (1, [2, 2, 2], [3, 3])
batch = batchify((x_1, x_2))

names = ['centers', 'contexts_negatives', 'masks', 'labels']
for name, data in zip(names, batch):
    print(name, '=', data)
```


<!--
We use the `batchify` function just defined to specify the minibatch reading method in the `DataLoader` instance.
-->

*dịch đoạn phía trên*

<!-- ===================== Kết thúc dịch Phần 4 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 5 ===================== -->

<!--
## Putting All Things Together
-->

## *dịch đoạn phía trên*


<!--
Last, we define the `load_data_ptb` function that read the PTB dataset and return the data iterator.
-->

*dịch đoạn phía trên*


```{.python .input  n=16}
#@save
def load_data_ptb(batch_size, max_window_size, num_noise_words):
    num_workers = d2l.get_dataloader_workers()
    sentences = read_ptb()
    vocab = d2l.Vocab(sentences, min_freq=10)
    subsampled = subsampling(sentences, vocab)
    corpus = [vocab[line] for line in subsampled]
    all_centers, all_contexts = get_centers_and_contexts(
        corpus, max_window_size)
    all_negatives = get_negatives(all_contexts, corpus, num_noise_words)
    dataset = gluon.data.ArrayDataset(
        all_centers, all_contexts, all_negatives)
    data_iter = gluon.data.DataLoader(dataset, batch_size, shuffle=True,
                                      batchify_fn=batchify,
                                      num_workers=num_workers)
    return data_iter, vocab
```


<!--
Let us print the first minibatch of the data iterator.
-->

*dịch đoạn phía trên*


```{.python .input  n=17}
data_iter, vocab = load_data_ptb(512, 5, 5)
for batch in data_iter:
    for name, data in zip(names, batch):
        print(name, 'shape:', data.shape)
    break
```


## Tóm tắt

<!--
* Subsampling attempts to minimize the impact of high-frequency words on the training of a word embedding model.
* We can pad examples of different lengths to create minibatches with examples of all the same length 
and use mask variables to distinguish between padding and non-padding elements, so that only non-padding elements participate in the calculation of the loss function.
-->

*dịch đoạn phía trên*


## Bài tập


<!--
We use the `batchify` function to specify the minibatch reading method in the `DataLoader` instance and print the shape of each variable in the first batch read.
How should these shapes be calculated?
-->

*dịch đoạn phía trên*


<!-- ===================== Kết thúc dịch Phần 5 ===================== -->
<!-- ========================================= REVISE PHẦN 2 - KẾT THÚC ===================================-->


## Thảo luận
* [Tiếng Anh - MXNet](https://discuss.d2l.ai/t/383)
* [Tiếng Việt](https://forum.machinelearningcoban.com/c/d2l)


## Những người thực hiện
Bản dịch trong trang này được thực hiện bởi:
<!--
Tác giả của mỗi Pull Request điền tên mình và tên những người review mà bạn thấy
hữu ích vào từng phần tương ứng. Mỗi dòng một tên, bắt đầu bằng dấu `*`.
Tên đầy đủ của các reviewer có thể được tìm thấy tại https://github.com/aivivn/d2l-vn/blob/master/docs/contributors_info.md
-->

* Đoàn Võ Duy Thanh
<!-- Phần 1 -->
* 

<!-- Phần 2 -->
* 

<!-- Phần 3 -->
* 

<!-- Phần 4 -->
* 

<!-- Phần 5 -->
* 

