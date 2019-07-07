---
layout: post
title: "Building an image generator for machine learning tasks using PYAutoGUI"
author: "Joakim Lönnegren"
categories: blog
tags: [Age of Empires 2, PYAutoGUI, FastAI]
comments: true
image:
  feature: aoe2-ml-image-generator/early-dark-age.jpg
  teaser: aoe2-ml-image-generator/early-dark-age-teaser.jpg
credit: The fast.ai team
creditlink: https://fast.ai
---

After starting the fastai machine learning course I wanted to do some experiments with a game I play and love, Age of Empires 2. The first thing you need for a computer vision task (which is the first lesson) is a couple of hundred training images. Instead of using Google Images I decided to test my GUI automation skills with pyautogui. Google would probably have taken much less time in the end but it was a fun project, here's how I did it.

The code is available on [Github](https://github.com/joalon/aoe2-ml-image-generator). To inspect it run the following:
```bash
git clone https://github.com/joalon/aoe2-ml-image-generator
cd aoe2-ml-image-generator
python3 -m venv venv
. venv/bin/activate
pip -r requirements.txt
```

Disclaimer: To actually run the project you'll need Age of Empires 2 purchased through steam (or another way to start it on linux, though you'll have to modify the start_aoe2 function in the file ).

With that out of the way, let's begin! Using pyautogui is pretty simple, I'm using arch linux, fish shell and python3 for the following code snippets.

Setup a project folder and a venv:

```bash
mkdir aoe2-ml-image-generator
cd aoe2-ml-image-generator
python3 -m venv venv
. venv/bin/activate.fish
pip install pyautogui     # You might have to install the scrot package from your regular package manager. Pyautogui relies on scrot for its screenshot functionality
```

Open up aoe2-image-gen.py. Here are some key pyautogui API functions:

```python
pyautogui.locateCenterOnScreen
pyautogui.click
pyautogui.typewrite
pyautogui.screenshot
```

When taking screenshots for the pyautogui.locateCenterOnScreen function I used spectacle and gimp. Any image manipulation software and screenshot tool would do. Here is how to start Age of Empires 2 as well as a function for clicking through to the map editor.

```python
def start_aoe2():
     subprocess.Popen(["bash", "-c", "steam steam://rungameid/221380"])
     wait_for_image('images/main-menu/aoe2-main-menu.png')

def wait_for_image(image):
    while pyautogui.locateOnScreen(image) == None:
        time.sleep(0.5)

def open_map_editor():
    map_editor_location_center = pyautogui.locateCenterOnScreen('images/main-menu/aoe2-map-editor-button.png')
    pyautogui.click(map_editor_location_center)

    create_scenario_button_center = pyautogui.locateCenterOnScreen('images/main-menu/aoe2-main-menu-create-scenario-button.png')
    pyautogui.click(create_scenario_button_center)

    create_button_center = pyautogui.locateCenterOnScreen('images/main-menu/aoe2-main-menu-create-button.png')
    pyautogui.click(create_button_center)

    wait_for_image('images/map-editor/aoe2-map-editor-main-starting-view.png')
```

The rest of the generation process I'll leave as an exercise for the reader.

After a couple of hours of work our program can now generate about twelve 224x224 pixel png:s of a villager or empty terrain per minute. Not precisely light speed. I'm still working on some bugs, for example here are some pictures where the villager placement is either off-screen or failed because of an obstacle:

![No villager in sight](/images/aoe2-ml-image-generator/no-villager-in-sight.png "No villager in sight")
![An offscreen villager](/images/aoe2-ml-image-generator/off-screen-villager.png "Where is it?")

And here are some where it placed a villager where there shouldn't be one:

![An unexpected villager](/images/aoe2-ml-image-generator/an-unexpected-villager.png "An unexpected villager")
![Another unexpected villager](/images/aoe2-ml-image-generator/an-unexpected-villager_2.png "Nobody expects...")

I let it stay on over night and got 2000 pictures for a dataset that you can download here: [joalon.se/datasets](https://joalon.se/datasets/villager-or-not.tgz). If you're following the fastai course you can download the dataset by doing `untar_data('https://joalon.se/datasets/villager-or-not')`
It still contains some misslabeled images but that'll get cleaned up during upcoming posts. Next time I'll use these images to train a model, stay tuned!
