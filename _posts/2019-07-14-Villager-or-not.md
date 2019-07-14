---
layout: post
title: "Villager or not - Single label classification using the fastai framework"
author: "Joakim Lönnegren"
categories: blog
tags: [Age of Empires 2, fastai]
image:
  feature: aoe2-villager-or-not/early-feudal-age.jpg
  teaser: aoe2-villager-or-not/early-feudal-age-teaser.jpg
---

Last post we created a dataset for a 'villager or not a villager' classification task. This time we'll train a model and see how well we can get a neural network to recognize said villager. We'll use a resnet34 model created with the fastai framework, the same as from lesson 1 in the fastai deep learning course 2019.

This code is on [Github](https://github.com/joalon/aoe2-villager-or-not). I did most of the work on a [Google Colab notebook](https://colab.research.google.com).

Here's the code for creating a neural net and training it on the dataset:

```python
from fastai.datasets import untar_data, download_data
from fastai.vision import ImageDataBunch, cnn_learner, get_image_files, imagenet_stats, models, error_rate
from google.colab import drive

drive.mount('/content/gdrive', force_remount=False)

path = untar_data('https://joalon.se/datasets/villager-or-not');
training_path = path/'training'
fnames = get_image_files(training_path)
pat = r'/([^/]+)_\d+.png$'

data = ImageDataBunch.from_name_re(training_path, pat=pat, fnames=fnames)
data.normalize(imagenet_stats)

learn = cnn_learner(data, models.resnet34, metrics=error_rate)

learn.unfreeze()
learn.fit_one_cycle(10)

learn.save('/content/gdrive/My Drive/Villager or not/first-villager-model')
```
Here you can see it running:
![Training](/images/aoe2-villager-or-not/training.png)

Note! I did have to add a GPU to the colab notebook, otherwise the training time for each epoch was 10 minutes instead of 10 seconds.

![Add GPU](/images/aoe2-villager-or-not/add-gpu-colab.png)

![Choose GPU](/images/aoe2-villager-or-not/choose-gpu-colab.png)

Let's see how it did, we can plot a confusion matrix:
```python
interp = ClassificationInterpretation.from_learner(learn)
interp.plot_confusion_matrix()
```
![Confusion matrix](/images/aoe2-villager-or-not/confusion-matrix.png)

That's cool, it's gotten 5 images wrong in the whole set. In the last post we concluded that the image generation was a bit flawed, now is a good time to check which images the model was most confused about.
```python
interp.plot_top_losses(9, figsize=(15,11))
```
![Top losses](/images/aoe2-villager-or-not/top-villager-losses.png)

Not very strange that these images have a high loss, they're clearly wrongly labelled. It's the perfect time to use a cool widget from the fastai library called the ImageCleaner to do some relabeling:
```python
ds, idxs = DatasetFormatter().from_toplosses(learn)
ImageCleaner(ds, idxs, path)
```
![Image cleaner](/images/aoe2-villager-or-not/image-cleaner.png)

I had to spend quite some time relabeling images. The generator wasn't as good as I expected, this model has probably overfit to be able to do so good on the wrongly labeled images.

![A water obstacle](/images/aoe2-villager-or-not/image-cleaner-water-obstacle.png)

Note: I couldn't run the ImageCleaner on Google Colab since it doesn't allow running widgets so, to run it, I had to use my laptop. Unfortunately I don't have a current GPU so I had to install pytorch as CPU only:

```bash
pip3 install https://download.pytorch.org/whl/cpu/torch-1.1.0-cp37-cp37m-linux_x86_64.whl
pip3 install https://download.pytorch.org/whl/cpu/torchvision-0.3.0-cp37-cp37m-linux_x86_64.whl
pip3 install fastai
```

[Pytorchs website](https://pytorch.org/get-started/locally/) has a really good tool for getting installation instructions for your own environment:
![CPU only pytorch](/images/aoe2-villager-or-not/pytorch-cpu-only.png)

The ImageCleaner widget doesn't actually change your dataset on disk, it creates a cleaned.csv file in the training path which can be loaded and retrained with:

```python
df = pd.read_csv(path/'cleaned.csv', header='infer')

db = (ImageList.from_df(df, path)
                   .split_none()
                   .label_from_df()
                   .databunch(bs=64))

learn = cnn_learner(db, models.resnet34, metrics=error_rate)
learn = learn.load('villager-or-not')
learn.unfreeze()
learn.fit_one_cycle(10)

learn.save('villager-or-not')
learn.export()
```
We'll need the export.pkl file from executing `learn.export()` to create a production model and run predictions on.

To finally test our model let's grab a couple of screenshots the model hasn't seen before to run inference with. I took some from the [AoE 2 Wiki](https://ageofempires.fandom.com/wiki/) and cropped them. The champion is a unit the model hasn't seen before, it'll be interesting to see how it handles that.

![Test image](/images/aoe2-villager-or-not/villager-test.png)
![Test image2](/images/aoe2-villager-or-not/gold-test.png)
![Test image3](/images/aoe2-villager-or-not/champion-test.png)

The directory you're running the following code in should contain the aforementioned export.pkl.

```python
from fastai.vision import load_learner, open_image
from PIL import Image

learn = load_learner('./')
image_names = ['villager-test.png', 'gold-test.png', 'champion-test.png']

for i in range(3):
    image = open_image(image_names[i])
    prediction = learn.predict(image)
    print(image_names[i] + ': ' + prediction[0])
```
![Prediction result](/images/aoe2-villager-or-not/result.png)

It works! Although it classified the champion as a villager it has never seen another unit so that's an honest mistake. Next lesson is multi label classification, stay tuned.