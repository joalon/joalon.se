---
layout: post
title: "Multi unit classification- Detecting units"
author: "Joakim Lönnegren"
categories: blog
tags: [Age of Empires 2, fastai]
image:
  feature: aoe2-multi-unit/feature.png
  teaser: aoe2-multi-unit/teaser.png
---
This time we're going to be assigning multiple labels to an image depending on the units appearing in it. The model will pretty much stay the same, what's different is the dataset and how we use it. For single label classification we created a dataset where the filename contained the label. Now we'll just call the images a number and use a csv file to hold the labels.

The dataset looks something like this:

```
multi-unit/
  labels.csv
  train/
    0.png
    1.png
    2.png
    etc...
```

Where labels.csv looks like this:
![csv show head](/images/aoe2-multi-unit/show-csv-head.png)


You can then use the dataset with the following code:

```
path = untar_data('https://joalon.se/datasets/multi-unit')
src = (ImageList.from_csv(path, 'labels.csv', folder='train', suffix='.png')
       .split_by_rand_pct(0.2)
       .label_from_df(label_delim=' '))

data = (src.transform(tfms, size=224)
        .databunch().normalize(imagenet_stats))

data.show_batch(rows=3, figsize=(15,10))
```
![Peeking at the data](/images/aoe2-multi-unit/data-show-batch.png)

This time we'll have to create some other metrics for accuracy. The model will return a probability that the image we're passing to it contains one of the labels, so we'll say that a probability of over 0.2 means the model predicted the label. Here's how to create the model with the metric:

```python
acc_02 = partial(accuracy_thresh, thresh=0.2)

learn = cnn_learner(data, models.resnet50, metrics=[acc_02])
```

![lr finder](/images/aoe2-multi-unit/learning-rate-finder.png)

Do note that metrics aren't used during training other than for showing information.

![Training time](/images/aoe2-multi-unit/fit-one-cycle.png)

With the first dataset of about 2000 images I generated I got some pretty bad results. The model thought that a scout cavalry was a penguin, for example. So I generated up to 10 000 images with substantially more of them just containing a single unit. I'm guessing it had trouble with it being too many units as well as them having different animations/facing different ways. With the expanded dataset I got much better results. Let's see the loss:

![plot losses](/images/aoe2-multi-unit/plot-losses.png)

As long as the training loss is higher than the validation loss we're not risking overfitting.

Now let's try to make a prediction. I took a picture of the berserk from the aoe2 wiki:

![Berserk](/images/aoe2-multi-unit/Berserk.png)

![Prediction time](/images/aoe2-multi-unit/berserk-predictions.png)

It does classify it as a berserk yet it's also pretty sure about the heavy swordsman. Let's do some others:

![Boyar](/images/aoe2-multi-unit/boyar.jpg)
![Boyar prediction](/images/aoe2-multi-unit/boyar-prediction.png)

![Chess](/images/aoe2-multi-unit/aoe2-chess.png)
![Chess prediction](/images/aoe2-multi-unit/aoe2-chess-predictions.png)

The model does reasonably well but the dataset might still be a bit too small. In the teutonic knight image it didn't recognize the villager, the knight or the monks and it wrongly predicted a legionary. The dataset used in the actual lesson was 32 GB of pictures of the amazon rainforest so still have some time to go before closing in on those numbers. Next part of lesson 2 is image segmentation, probably my most anticipated lesson!