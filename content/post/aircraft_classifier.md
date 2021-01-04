---
title: "Image recognition with PyTorch and fastai"
summary: "Computer vision is one of the most fascinating domains in Machine Learning. Libraries like PyTorch and more recently, fastai, have made these kinds of models extraordinarily accessible. In this post, we build an aircraft classifier from gathering data to training and deployment."
date: 2020-12-22
tags: ["computer vision", "transfer learning", "pre-trained models", "deployment", "pytorch", "torchvision", "fastai", "fast.ai", "python"]
draft: false
---

*Test my [aircraft classifier](https://aircraft-classifier-007.herokuapp.com/) with your own images*  

### *fastai*

The content of this article is inspired by the book [*Deep Learning for Coders with fastai & PyTorch*](https://github.com/fastai/fastbook), written by Jeremy Howard, founder/CEO of half a dozen very successful tech companies like *Kaggle*, and fellow citizen Sylvain Gugger, mathematician and research engineer at *Hugging Face*:

![fastbook](/res/aircraft_classifier/fastbook.resized.jpg)

Released in 2020, this book is one of the good things that actually happened this year. It is both beginner friendly and very complete --- 547 pages without the appendices --- from a practitioner's point of view. More importantly, it is very enjoyable to read and code with. As for [*fastai*](https://docs.fast.ai/), it is a Deep Learning Python module [1] that is built on top of *PyTorch*. It aims at popularizing, facilitating and speeding up the process of building Deep Learning models and applications. Spoiler alert: in my opinion, it does an incredible job at achieving these goals. Let us see how to build an image recognition model with *fastai*.

### Building a Deep Learning model

The goal of the classifier is to distinguish between an **airliner**, a **fighter jet** and an **attack helicopter**. The first step is to gather relevant pictures and to build an appropriate dataset.

#### Create the dataset

We will use Azure's Bing Image Search API in order to collect pictures of aircrafts. It lets the user retrieve a maximum of 150 results per query. A free Azure account is needed for this part. Once the key for the Bing Image Search resource is generated, *fastai*'s function ```utils.search_images_bing()``` makes it very easy to get the desired pictures in properly organized folders:

```python
# List the keywords to be queried and define a folder to retrieve the matching pictures
flying_types = 'mirage 2000','apache helicopter','airbus a380'
path = Path('flying_things')

# Collect and store the pictures, if not already done
if not path.exists():
    path.mkdir()
    for o in flying_types:
        dest = (path/o)
        dest.mkdir(exist_ok=True)
        results = search_images_bing(key, f'flying thing: {o}')
        download_images(dest, urls=results.attrgot('contentUrl'))

# Remove corrupted files
failed = verify_images(fns)
failed.map(Path.unlink)
```

*Due to recent changes in the Bing API, the native function ```utils.search_images_bing()``` is broken. A working version can be found in the [notebook](https://github.com/datatrigger/computer_vision).*

Notice that we did not exactly searched for "arliner" / "fighter jet" / "attack helicopter". Querying specific models gives more relevant pictures in my experience. Now that we have images, we build the dataset using the class ```DataBlock()```:

```python
# Define the rules to form the dataset
flying_things = DataBlock(
    # Specify the types of the independent / dependent variables
    blocks=(ImageBlock, CategoryBlock),
    # get_image_files() recursively retrieves images from a given path
    get_items=get_image_files,
    # Split the dataset into train/validation subsets
    splitter=RandomSplitter(valid_pct=0.2, seed=42),
    # The label of a given picture is the name of its containing folder
    get_y=parent_label,
    # Resize pics to 224px*224px, and randomly crop them to half their original size
    item_tfms=RandomResizedCrop(224, min_scale=0.5),
    # Apply standard data augmentation techniques:
    # rotation, flipping, perspective warping, contrast/brightness changes...
    batch_tfms=aug_transforms()
)

# Make the actual dataset
dls = flying_things.dataloaders(path)
```

And it is done!

```python
# Display 16 pictures from the validation set
dls.valid.show_batch(max_n=16, nrows=4)
```

![aircraft_results](/res/aircraft_classifier/validation_set_sample.png)

We can already see that a few images are irrelevant : pictures taken from inside the cockpit, cartoons, video games, aircrafts far away, etc...

#### Train the model **and** clean the data

We are going to build the model incrementally, that is use our model to clean the data, improve the model with purged data and so on. Below, the pre-trained 18-layer convolutional neural network [ResNet](https://arxiv.org/pdf/1512.03385.pdf) is loaded. it is available in the ```torchvision``` library from PyTorch. It has already been fed 1.3 million images. We adapt it for our specific problem by training it with our own dataset.

```python
# Reload the data the second time, after the first cleaning:
# dls = flying_things.dataloaders(path)
learn = cnn_learner(dls, resnet18, metrics=error_rate)
learn.fine_tune(4)
```

![epochs](/res/aircraft_classifier/epochs.png)

4 epochs appears to be enough as the loss is not decreasing anymore at this point. Now let us use the GUI ```ImageClassifierCleaner()``` to clean the dataset. It shows the pictures with the worst losses, and lets the user either erase them or change their label if it is wrong. More precisely, it returns the indices of the said images.

```python
cleaner = ImageClassifierCleaner(learn)
cleaner
```

![data cleaner](/res/aircraft_classifier/cleaner.png)

```python
# Use the indices returned by `ImageClassifierCleaner()` to do the actual cleaning.
for idx in cleaner.delete(): cleaner.fns[idx].unlink()
for idx,cat in cleaner.change(): shutil.move(str(cleaner.fns[idx]), path/cat)
```

The process of training the model and cleaning the data is repeated until the accuracy and confusion matrix look satisfactory:

```python
# Build the confusion matrix
interp = ClassificationInterpretation.from_learner(learn)
interp.plot_confusion_matrix()
```

![confusion matrix](/res/aircraft_classifier/confusion_matrix.png)

Only four aircrafts are misclassified by the final model. We can show the images with the highest loss:

```python
# Plot the pictures with highest losses
interp.plot_top_losses(5, nrows=5)
```

![highest losses](/res/aircraft_classifier/highest_losses.png)

We decide to stop the experiment at this point. The error rate on the validation set is about 6 %. We can now save the model with ```Learner.export()```. We will be able to use it later with ```Learner.load_learner()```.

#### Deployment

Let us try our aircraft classifier out. For a given image, ```Learner.predict()``` generates the predicted category, its index and probabilities associated with each category:

```python
learn.predict('images/mirage_2000.jpg')
```

![prediction for a mirage 2000](/res/aircraft_classifier/predict.png)

The labels are ordered this way:

```python
learn.dls.vocab
```

![labels](/res/aircraft_classifier/labels.png)

The model is ready to be deployed on a cloud platform. In order to let any user test our aircraft classifier, we have built a very simple [web application](https://aircraft-classifier-007.herokuapp.com/) using IPython widgets (GUI components) and Voilà, a library that turns Jupyter notebooks into standalone application. The app is currently deployed on *Heroku*:

![web app](/res/aircraft_classifier/web_app.png)

### Source code

* [Notebook](https://github.com/datatrigger/computer_vision) to build the dataset and the model.
* [Source of the Heroku app](https://github.com/datatrigger/aircraft_classifier) where the aircraft classifier is deployed.

### Conclusion

In this post, we have built an image recognition model showing very good performance, from data collection to training, fine-tuning, testing and deploying. And we were able to do so in less than 50 lines of code. Therefore, I reckon that *fastai* is an extremely efficient API for Deep Learning applications. As it is built on top of *PyTorch*, it does not prevent the user to work with native features as well. Besides, the source code of fast.ai is very intelligible, hence easily customizable. As a Deep Learning practitioner, *fastai* has come to be one of my main tools in addition to plain PyTorch.

### References

[[1]](https://arxiv.org/abs/2002.04688) *fastai: A Layered API for Deep Learning* , Jeremy Howard, Sylvain Gugger. fast.ai, University of San Francisco, 2020.
