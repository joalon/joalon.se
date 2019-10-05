---
layout: post
title: "Running the AoE 2 map editor headless"
author: "Joakim Lönnegren"
categories: blog
tags: [Age of Empires 2, Xvfb, Python, PyAutoGUI]
image:
  feature: headless-map-editor/feature.png
  teaser: headless-map-editor/teaser.png
---

I've been running the AoE2 map editor with a pyautogui script to generate machine learning datasets. A problem is this hogs the mouse and screen on my laptop for long periods of time. It would be nice to be able to run this in the background. On Linux and using the X graphics server there are several ways to do this, I'll be using PyVirtualDisplay which will create a nested/virtual X server. The PyVirtualDisplay is a python wrapper that integrates with Xvfb, Xvnc and Xephyr but it requires modifications of the python script itself to start.

As usual you can find the code on [Github](https://github.com/joalon/aoe2-ml-image-generator). This time I'm building on top of the earlier work on the aoe2-ml-image-generator repository.

Prerequisites
In this post I'll use the following list of applications,
* xwd
* xwud
* imagemagick
* glxgears (included as a demo application in OpenGL. Optional)
* Xvfb
* Xephyr
* python 3 & pip
* Steam and Age of Empires 2

On Arch you can install most of these with `sudo pacman -S xorg-xwd xorg-xwud xorg-server-xvfb imagemagick python3 python-pip`. Glxgears can just as well be substituted with any other graphical program if you don't have it installed since setting up OpenGL/VirtualGL is out of scope for this post.

The code is available on my [Github](https://github.com/joalon/aoe2-ml-image-generator).

## Virtual Frame Buffer
You can try the Xvfb out with the following commands

```fish
Xvfb :1 -screen 0 1920x1080x24 > /dev/null &
env DISPLAY=:1 glxgears > /dev/null &
xwd -root -display :1 | xwud
```

A window should now pop up with a picture of the famous glxgears.

![First Xvfb screenshot](/images/headless-map-editor/headless-glxgears.png)

To kill glxgears and xvfb you can run `fg` and `Ctrl-C` twice. This brings background processes started with & to the foreground so you can interrupt them. 

## Pyvirtualdisplay
[The PyVirtualDisplay](https://pypi.org/project/PyVirtualDisplay/) is a wrapper around Xvfb, Xephyr and Xvnc. I'll have to scrap the `start_aoe2` function and instead use EasyProcess to start with a virtual display:

```python
from easyprocess import EasyProcess
from pyvirtualdisplay.smartdisplay import SmartDisplay

def generate_villager_dataset(numberOfImages)
     with SmartDisplay(visible=1 if VISIBLE else 0, size=(1024,768)) as disp:
         pyautogui._pyautogui_x11._display = Xlib.display.Display(os.environ['DISPLAY'])
         with EasyProcess('bash -c "steam steam://rungameid/221380"'):
           
             ### Rest of the code here ###
```

If you add the preceding code to the python script and then check which processes it starts, it will start either Xvfb or Xephyr, depending on the global `VISIBLE=True/False` variable.

Setting the display for pyautogui on line 2 in the function is a workaround to send mouse clicks and keyboard presses to the right context. Here's a [stackoverflow answer](https://stackoverflow.com/questions/35798478/how-i-can-attach-the-mouse-movement-pyautogui-to-pyvirtualdisplay-with-seleniu) with more details. This might get fixed in the future, you can follow the problem and development of the headless/remote functionality in issue [133](https://github.com/asweigart/pyautogui/issues/133). Also note the Display `size=(1920, 1080)` and `--resolution=1920x1080`, Age of Empires 2 won't run with a resolution smaller than 960\*600. I found using another resolution than my native would make it hard for pyautogui to recognize the images when using `pyautogui.locateCenterOnScreen`

![Resolution error](/images/headless-map-editor/aoe2-resolution-error.png)

## Python Argparse
The python script doesn't need to change when using Xvfb, however I'll import the argparse library and add some flags to the script to improve usability. I used the tips in [this Stackoverflow question](https://stackoverflow.com/questions/27529610/call-function-based-on-argparse) for calling functions via a function map from command line arguments. Note that the argument parsing code must be located after any functions they call otherwise you'll get a "NameError: name ... is not defined".

```python
import argparse

### Rest of the code here ###


FUNCTION_MAP = {'version' : run_version,
                'map_editor' : run_map_editor,
                'villagers' : generate_villager_dataset,
                'multi_label' : generate_multi_label_dataset }

parser = argparse.ArgumentParser(description='Generate machine learning datasets using the Age of Empires 2 map editor running under steam.')
parser.add_argument('command', choices=FUNCTION_MAP.keys())
parser.add_argument('-n', type=int, nargs=1, default=[5], help='Number of images to generate in the dataset.')
parser.add_argument('-v', '--visible', action='store_true', default=False, help='Start in a visible window, otherwise it runs in a virtual frame buffer.')

args = parser.parse_args()

VISIBLE = args.visible

argument_function = FUNCTION_MAP[args.command]
argument_function(numberOfImages=args.n[0])
```

Running this I can now take a screenshot of the map editor without running it on the main display server using the snippets from the last code block. Running `python3 aoe2-ml-image-generator.py --visible multi_label` should start the map editor in a new window running under a separate X11 server and start generating images from the last lesson and pt them under `results/`

![Headless map editor](/images/headless-map-editor/headless-map-editor.png)

Before I could start steam under a new X server I had to install libxnvctrl (`sudo pacman -S libxnvctrl`), otherwise I would get the error "Error: can't open display" when I tried to start steam in Xephyr or Xvfb. I believe this has to do with my laptop having an Nvidia card. You might have to set this up differently on your own hardware. You might also have to fiddle a bit with steam command line options when starting AoE2. I had to specify the resolution since it won't automatically pick it up when running in a virtual display. Here's a handy reference on [valvesoftware.com](https://developer.valvesoftware.com/wiki/Command_Line_Options).

I've run these tests with the 1024 \* 768 resolution since the graphics seem to cut off the edges any further. You can still see this in the following screenshot

![Graphics cutoff](/images/headless-map-editor/graphics-cutoff.png)

I'm not sure why this cut off occurs but I'd guess it has to do with the nested X server either not being able to run on the graphics card or some other configuration issue. Right now 1024 \* 768 will do for me.

Thanks for reading! I tried to get this working in a docker container but alas! Steam is hard to containerize in a nice manner, this will have do for now. Hope this was useful for automating headless X applications with PyAutoGUI and Xvfb. Since Wayland is fast replacing X11 I'll look into porting this workflow to wayland applications in the future.
