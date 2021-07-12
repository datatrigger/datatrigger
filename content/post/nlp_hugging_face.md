---
title: "NLP with 🤗 Hugging Face"
summary: "Zero-shot classification is basically text classification with no training at all. How does it compare with transfer learning/fine-tuning? We'll see using the beloved 🤗 `transformers` library."
date: 2021-07-12
tags: ["NLP", "zero-shot classification", "text classification", "distilbert", "transformers", "hugging face"]
draft: false
---

*Full notebook available on [GitHub](https://github.com/datatrigger/nlp_hugging_face)*  

## Text Classification: Transfer Learning vs Zero-Shot Classifier

🤗 [Hugging Face](https://huggingface.co/) is, in my opinion, one of the best things that has happened to Data Science over the past few years. From generalizing access to state-of-the-art NLP models with the [`transformers`](https://huggingface.co/transformers/) library to [distillation [1]](https://arxiv.org/abs/1910.01108), they are having a huge impact on the field. I recently found out about "Zero-Shot Classification". These models are classifiers that do not need any fine-tuning, apart from being told which classes it should predict. They are built on top of Natural Language Inference models, whose task is determining if sentence *A* implies, contradicts or has nothing to do with sentence *B*. This excellent [blog post](https://joeddav.github.io/blog/2020/05/29/ZSL.html) written by 🤗 Hugging Face researcher Joe Davison provides more in-depth explanations.  
  
Here is an example:

```python
# transformers 3.5.1 in this notebook
from transformers import pipeline

# By default, the pipeline runs on the CPU (device=-1). Set device to 0 to use the GPU (and to 1 for the second GPU, etc...)
classifier = pipeline("zero-shot-classification", device=0)
classifier(
    "Parallel processing with GPUs is the savior of Deep Learning",
    candidate_labels=["education", "politics", "technology"],
)
```

![zero-shot classification example](/res/nlp_hugging_face/1.zs_classification_example.png)

The classifier guessed that the sentence is about tech with a probability over 99%. **But how does Zero-Shot classification compare with plain "old" fine-tuned text classification?**

### I) BBC News dataset

Let's build a classifier of news articles labeled *business*, *entertainment*, *politics*, *sport* and *tech*. Available [here](http://mlg.ucd.ie/datasets/bbc.html), the dataset consists of 2225 documents from the BBC news website from the years 2004/2005. It was originally built for a Machine Learning paper about clustering [[2]](http://mlg.ucd.ie/files/publications/greene06icml.pdf).  
  
Articles are individual .txt files spread into 5 folders, one for each folder. The listing below puts articles/labels into a `pandas.DataFrame()`.

```python
# Utilities to handle directories, files, paths, etc...
from os import listdir
from os.path import isdir, isfile, join
from pathlib import Path

# Most original import ever
import pandas as pd

path_to_bbc_articles="bbc"
labels=[] # labels for the text classification
label_dataframes=[] # for each label, get the articles into a dataframe

for label in [dir for dir in listdir(path_to_bbc_articles) if isdir(join(path_to_bbc_articles, dir)) and dir!=".ipynb_checkpoints"]:
    labels.append(label)
    label_path=join(path_to_bbc_articles, label)
    articles_list=[]
    for article_file in [file for file in listdir(label_path) if isfile(join(label_path, file))]:
        article_path=join(label_path, article_file)
        article=Path(article_path).read_text(encoding="ISO-8859-1") # Tried utf-8 (of course) but encountered error
        # Stackoverflow said "try ISO-8859-1", it worked (dataset is 11 years old)
        articles_list.append(article)
    label_dataframes.append(pd.DataFrame({'label': label, 'article': articles_list}))
    
df=pd.concat(label_dataframes, ignore_index=True) # Concatenate all the dataframes
```

```python
# Number of articles per label
df.value_counts('label')
```

![number of articles per label](/res/nlp_hugging_face/2.value_counts.png)

We will need integer labels to feed the transformer model:

```python
df['label_int']=df['label'].apply(lambda x:labels.index(x))
```

Here are 5 random rows from the final dataframe:

![a few rows extracted from the dataframe](/res/nlp_hugging_face/3.df_sample.png)

### II) Fine-tuning a pretrained text classifier

After building the train/validation/test sets, we will go straight the point by using the [`DistilBERT`](https://huggingface.co/transformers/model_doc/distilbert.html) pre-trained transformer model (and its tokenizer).

> *It is a small, fast, cheap and light Transformer model trained by distilling BERT base. It has 40% less parameters than bert-base-uncased, runs 60% faster while preserving over 95% of BERT’s performances.* 

```python
# Train set, validation set and test set
from sklearn.model_selection import train_test_split

train_val, test = train_test_split(df, test_size=0.1, random_state=42, shuffle=True)
train, val = train_test_split(train_val, test_size=0.2, random_state=42, shuffle=True)

# Reset the indexes of the 3 pandas.DataFrame()
train, val, test = map(lambda x:x.reset_index(drop=True), [train, val, test])
```

#### Tokenize

Loading DistilBERT's tokenizer, we can see that this transformer model takes input sequences composed of up to 512 tokens:

```python
# Load Distilbert's tokenizer
from transformers import DistilBertTokenizerFast
tokenizer = DistilBertTokenizerFast.from_pretrained('distilbert-base-uncased')
tokenizer.max_model_input_sizes
```

![DistilBERT max_input_size](/res/nlp_hugging_face/4.distilbert_max_input_size.png)

How does this compare with the lengths of the tokenized BBC articles?

```python
tokenized_articles_lengths=pd.DataFrame({'length': list(map(len, tokenizer(df['article'].to_list(), truncation=False, padding=False)['input_ids']))})
tokenized_articles_lengths.describe()
```

![Length of the tokenized articles](/res/nlp_hugging_face/5.tokenized_articles_lengths.png)

The articles are, on average, 488-token-long. The longest news is composed of 5303 tokens. This means that an important part of the articles will be truncated before being fed to the transformer model. Here is the distribution of the lengths:

```python
import matplotlib.pyplot as plt
import seaborn as sns

# Fast less good-looking plot
# ax=sns.histplot(tokenized_articles_lengths)
# ax.set(xlabel='Length of tokenized articles', ylabel='Count', xlim=(0, 1200), title='Distribution of the # tokenized articles lengths')
# plt.show()

fig, ax = plt.subplots(figsize=(16, 16))
ax=sns.histplot(tokenized_articles_lengths, palette='dark')
ax.set(xlim=(0, 1200))
ax.set_xticks(range(0, 1200, 100))
ax.set_title('Distribution of the tokenized articles lengths', fontsize=24, pad=20)
ax.set_xlabel('Length of tokenized articles', fontsize = 18, labelpad = 10)
ax.set_ylabel('Count', fontsize = 18, labelpad = 10)
ax.tick_params(labelsize=14)
plt.savefig('tokenized_articles_length_distribution.png', bbox_inches='tight');
```

![Distribution of the article lengths](/res/nlp_hugging_face/6.tokenized_articles_length_distribution.png)

```python
from scipy.stats import percentileofscore
print(f'Percentile of length=512: {int(percentileofscore(tokenized_articles_lengths["length"],512))}th')
```

![Percentile of the 512-token limit](/res/nlp_hugging_face/7.percentile.png)

About 36% of the articles will be truncated to fit the 512-token limit of DistilBERT. The truncation is mandatory, otherwise the model crashes. We will use fixed padding for the sake of simplicity here.

#### Fine-tune DistilBERT

The train/validation/test sets must be procesμsed to work with either PyTorch or TensorFlow.

```python
# Format the train/validation/test sets
train_encodings = tokenizer(train['article'].to_list(), truncation=True, padding=True)
val_encodings = tokenizer(val['article'].to_list(), truncation=True, padding=True)
test_encodings = tokenizer(test['article'].to_list(), truncation=True, padding=True)

import torch

class BBC_Dataset(torch.utils.data.Dataset):
    def __init__(self, encodings, labels):
        self.encodings = encodings
        self.labels = labels

    def __getitem__(self, idx):
        item = {key: torch.tensor(val[idx]) for key, val in self.encodings.items()}
        item['labels'] = torch.tensor(self.labels[idx])
        return item

    def __len__(self):
        return len(self.labels)

train_dataset = BBC_Dataset(train_encodings, train['label_int'].to_list())
val_dataset = BBC_Dataset(val_encodings, val['label_int'].to_list())
test_dataset = BBC_Dataset(test_encodings, test['label_int'].to_list())
```

```python
# Fine-tuning
from transformers import DistilBertForSequenceClassification, Trainer, TrainingArguments

training_args = TrainingArguments(
    output_dir='./results',
    num_train_epochs=3,
    per_device_train_batch_size=8,
    per_device_eval_batch_size=4,
    weight_decay=0.01,
)

# The number of predicted labels must be specified with num_labels
# .to('cuda') to do the training on the GPU
model = DistilBertForSequenceClassification.from_pretrained("distilbert-base-uncased", num_labels=len(labels)).to('cuda')

trainer = Trainer(
    model=model,                         # the instantiated 🤗 Transformers model to be trained
    args=training_args,                  # training arguments, defined above
    train_dataset=train_dataset,         # training dataset
    eval_dataset=val_dataset             # evaluation dataset
)
```

```python
trainer.train()
trainer.save_model("bbc_news_model")
```

```python
# Generate predictions for the test set
predictions=trainer.predict(test_dataset)
```

#### Accuracy

```python
test_results=test.copy(deep=True)
test_results["label_int_pred_transfer_learning"]=predictions.label_ids
test_results['label_pred_transfer_learning']=test_results['label_int_pred_transfer_learning'].apply(lambda x:labels[x])
```

Now the following command prints an empty `pandas.DataFrame()`:

```python
test_results[test_results["label"]!=test_results["label_pred_transfer_learning"]].head()
```

The accuracy of the fine-tuned DistilBERT transformer model on the test set is **100%**!

### III) Zero-Shot Classification

We'll use the appropriate [`transformers.pipeline`](https://huggingface.co/transformers/main_classes/pipelines.html) to compute the predicted class for each article.

```python
from transformers import pipeline
classifier = pipeline("zero-shot-classification", device=0) # device=0 means GPU

# Compute the predicted label for each article
test_results['label_pred_zero_shot']=test_results['article'].apply(lambda x:classifier(x, candidate_labels=labels)['labels'][0])

# Reorder columns and save results
test_results=test_results[['article', 'label', 'label_pred_transfer_learning', 'label_pred_zero_shot']]
test_results.to_parquet("test_results.parquet")

# Accuracy
error_rate=len(test_results[test_results["label"]!=test_results["label_pred_zero_shot"]])/len(test_results)
print(f'Accuracy of the Zero-Shot classifier: {round(100*(1-error_rate), 2)} %')
```

![Zero-Shot Classification accuracy](/res/nlp_hugging_face/8.accuracy_zsc.png)

The Zero-Shot classifier does a really bad job compared with the fine-tuned model. However, given the number of labels &mdash; 5 &mdash; this result is not that catastrophic. It is well above the 20% a random classifier would achieve (assuming balanced classes). Glancing at a few random articles uncorrectly labeled by the Zero-Shot classifier, there does not seem to be a particularly problematic class, although such a assertion would require further investigation. But the length of the news could lead to poor performance. We can read about this on the [🤗 Hugging Face forum](https://discuss.huggingface.co/t/new-pipeline-for-zero-shot-text-classification/681/85). Joe Davison, 🤗 Hugging Face developer and creator of the Zero-Shot pipeline, says the following:

> *For long documents, I don’t think there’s an ideal solution right now. If truncation isn’t satisfactory, then the best thing you can do is probably split the document into smaller segments and ensemble the scores somehow.*

We'll try another solution: summarizing the article first, then Zero-Shot classifying it.

### IV) Summarization + Zero-Shot Classification

The easiest way to do this would have been to line up the `SummarizationPipeline` with the `ZeroShotClassificationPipeline`. This is not possible, at least with my version of the `transformers` library (3.5.1). The reason for this is that the `SummarizationPipeline` uses Facebook's BART model, whose maximal input length is 1024 tokens. However, `transformers`'s tokenizers, including `BartTokenizer`, do not automatically truncate sequences to the max input length of the corresponding model. As a consequence, the `SummarizationPipeline` crashes whenever sequences longer than 1024 tokens are given as inputs. Since there are quite a few long articles in the BBC dataset, we will have to make a custom summarization pipeline that truncates news longers than 1024 tokens.

```python
# Import the tokenizer and model for summarization (the same that are used by default in Hugging Face's summarization pipeline)
from transformers import BartTokenizer, BartForConditionalGeneration, BartConfig

model_bart = BartForConditionalGeneration.from_pretrained('facebook/bart-large-cnn').to('cuda') # Run on the GPU
tokenizer_bart = BartTokenizer.from_pretrained('facebook/bart-large-cnn')

# Custom summarization pipeline (to handle long articles)
def summarize(text):
    # Tokenize and truncate
    inputs = tokenizer_bart([text], truncation=True, max_length=1024, return_tensors='pt').to('cuda')
    # Generate summary between 10 (by default) and 50 characters
    summary_ids = model_bart.generate(inputs['input_ids'], num_beams=4, max_length=50, early_stopping=True)
    # Untokenize
    return([tokenizer_bart.decode(g, skip_special_tokens=True, clean_up_tokenization_spaces=False) for g in summary_ids][0])

# Apply summarization then zero-shot classification to the test set
test_results['label_pred_sum_zs']=test_results['article'].apply(lambda x:classifier(summarize(x), candidate_labels=labels)['labels'][0])

test_results.to_parquet("test_results.parquet")

error_rate_sum_zs=len(test_results[test_results["label"]!=test_results["label_pred_sum_zs"]])/len(test_results)
print(f'Accuracy of the Summmarization+Zero-Shot classifier pipeline: {round(100*(1-error_rate_sum_zs), 2)} %')
```

![Summarization + Zero-Shot Classification accuracy](/res/nlp_hugging_face/10.accuracy_szsc.png)

Adding the summarization before the zero-shot classification, **the accuracy jumped by ~23%**! Let us remember that there was no training whatsoever. From this perspective, a 78% accuracy looks pretty good to me! This result could probably be enhanced by tuning the summarizer's parameters regarding beam search or maximal length.

### V) Conclusion

Text classification is a piece of cake using 🤗 Hugging Face's pre-trained models: fine-tuning DistilBERT is fast (using a GPU), easy and it resulted in a 100% accuracy on the BBC News test set. Although this result should be confirmed with other train-test split (only 56 articles in the test set), it is absolutely remarkable. The raw Zero-Shot Classification pipeline from the `transformers` library could not compete at all with such a performance, ending up with a ~55% accuracy on the same test set. Nonetheless, this result is still decent considering the complete absence of training required by this method.  
  
Given the substantial length of the BBC News articles, we tried summarizing them before performing the Zero-Shot classification, still using the beloved `transformers` library. This method resulted in a +23% increase of accuracy. Another way would have been to carry out sentence segmentation before the Zero-Shot classification, and averaging the prediction over all an article's sentences.

We end up with two text classifiers:
* One that requires training and yields a 100% accuracy
* One that does not require any training, but yields a ~78% accuracy

Either way, way to go 🤗 Hugging Face!

## References

[[1]](https://arxiv.org/abs/1910.01108) Victor Sanh, Lysandre Debut, Julien Chaumond, Thomas Wolf. *DistilBERT, a distilled version of BERT: smaller, faster, cheaper and lighter.* (Hugging Face, 2020)  
  
[[2]](http://mlg.ucd.ie/files/publications/greene06icml.pdf) D. Greene and P. Cunningham. "Practical Solutions to the Problem of Diagonal Dominance in Kernel Document Clustering", Proc. ICML 2006.



