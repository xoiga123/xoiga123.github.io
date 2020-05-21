---
layout: post
title: Little mirror program
---

# 09/05/2020 - The mirror

### Recommended music: Nurture - Porter Robinson

My friend asked me to make a mirror program with the following specs:
- Running in the background.
- Capturing certain hotkeys that the user chose.
- Screen capturing, flipping the image, showing it and then closing, waiting for next screen capture.
- Written in any language I like.
- Cross-platform.

Fun fact: Artists in the old times usually have a mirror in their room, so they can see their work in a "mirrored" perspective and
look at it in a different way, improving their work in ways beyond my comprehension. Or so I've heard.

Seems pretty simple, right? I immediately took the job, since I've worked with OpenCV using python before. After a few hours of googling around and
putting together ~~other people's~~ my code, and *a few more days* lazying around, I kinda achieved all the goals.

```python
import numpy as np
import pyautogui
import cv2
import keyboard

events = [i for i in dir(cv2) if 'EVENT' in i]
print( events )

print("Please enter the keys you wanna use to capture screen.")
print("Type 'done' when completed.")
print("Shift and Ctrl can't be recorded, so just type in 'shift' or 'ctrl' if you want to use them in combination with other keys.")
print("Example: to use ctrl + q, type ctrl, then next line type q, then next line type done.")
print("Im not doing any input checking so if the program breaks, thats on you, and just restart the program.")

keys = ""
while True:
    inp = input("Type a key ('done' to confirm): ")
    if inp == "done": break
    keys += inp + "+"
keys = keys[:-1]
print("Keys recorded, to capture screen use " + keys)

scale = int(input("Enter scale percent to resize the image (100 is the original): "))
print("To close the image, press any key, or escape key to exit the program.")
print("To exit the program, either close the cmd, or press escape key when an image is showing.")

while True:
    keyboard.wait(keys)
    image = pyautogui.screenshot()
    image = cv2.cvtColor(np.array(image), cv2.COLOR_RGB2BGR)
    flipHorizontal = cv2.flip(image, 1)
    width = int(flipHorizontal.shape[1] * scale / 100)
    height = int(flipHorizontal.shape[0] * scale / 100)
    dim = (width, height)
    resized = cv2.resize(flipHorizontal, dim)
    cv2.imshow("Mirrored", resized)
    k = cv2.waitKey(0)
    #print(k)
    cv2.destroyAllWindows()
    if k == 27:
        break
```

Would you look at that! I had made something useful for the first time in my useless life. Freezing it up with PyInstaller, and I got a
*small* exe file of only ~50MBs. Apparently, when you import a library into your python file and freeze it, it take the whole library
with you. I know, crazy stuffs, right? I looked around for ways to reduce library size, like does `from .. import ..` take less space
than `import`?

<blockquote class="twitter-tweet"><p lang="en" dir="ltr"><a href="https://twitter.com/hashtag/python?src=hash&amp;ref_src=twsrc%5Etfw">#py
thon</a> tip: from-imports don&#39;t save memory. They execute and cache the entire module just like a regular import.</p>&mdash; Raymond 
Hettinger (@raymondh) <a href="https://twitter.com/raymondh/status/620993578118967296?ref_src=twsrc%5Etfw">July 14, 2015</a></blockquote> 
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

According to a core python dev, it doesn't. Great. Then how about taking some functions out of these libraries? Sounds good, doesn't work.
Too many dependencies. Okay, I've found others claming that cx_freeze or py2exe does a better job than PyInstaller, but with more difficult
syntaxes than the one-liner of PyInstaller. I didn't even consider this option, because my py files can be squeezed out as much as it could be,
the problem remains that my file contains 2 big ass libraries (opencv and numpy) that I only use a few lines of. Time to dig straight down.

To capture user hotkey while the exe is not in focused, the keyboard lib to hook keyboard presses is a must. Besides, it's somewhat light,
so I don't need to try hard and find another lib that does the same but lighter. Too much hassle.

I've used pyautogui to take a screenshot, and then opencv + numpy to convert into an image then flip, and opencv again to display and close.
That's almost like 3 libraries for 3 tasks. Opencv has to go with numpy, and do I really need them? To the trash bin you go. Reading the docs
of pyautogui, I've learned that they actually use PIL (Python Imaging Library) to capture the image. So I went with PIL instead of pyautogui.
You did your best, don't be ashamed.

# 18/05/2020 The overhaul

### Recommended music: X - Ed Sheeran

My friend asked me for another thing, which is zooming in and out of the mirrored image. Okay, will do. Let's just continue working
on the specs first. From 4 down to 2 libraries, I plan on a complete overhaul.

```python
from PIL import Image, ImageGrab
import keyboard

keyboard.wait("ctrl+q")
im = ImageGrab.grab(include_layered_windows=True)
im = im.transpose(Image.FLIP_LEFT_RIGHT)
im.show()
```

Look how clean that is. Like an angel banishing the wreck I made several days ago *I excluded the input of hotkeys though*.
Sadly, the image viewer does pop up, but there is no image. Hello Stackoverflow, I have come to you again.

> show() tries to execute the default image viewer with a start /wait command on a temporary image file. The /wait parameter is supposed to wait until the viewer exits, so that the file can be deleted. Unfortunately the default viewer under Vista and Windows 7 does not respond properly to /wait and return even before they've opened the file; the file gets deleted before it can be displayed.
> The usual fix is to edit ImageShow.py in the PIL package and add an extra command to wait a few seconds before deleting the file. This is a kludge, but it works most of the time.

Digging into the lib, what I have to put in is 
```
'start "Pillow" /WAIT "%s" '
"&& ping -n 2 127.0.0.1 >NUL "
'&& del /f "%s"' % (file, file)
```
Let's break that down, shall we? First, the program capture the image, then saves it to a tmp folder and attemps to open it.
The address of that temporary image file is `file`. `start`, well, starts a program and **supposedly** giving it a handle called
"Pillow" to open the image. Then, it does a hacky hacky trick. It pings the localhost, which is always available, two times.
The time between each pings is one second. Then it outputs that ping to NUL in order to delete it. This creates a timeout so the image viewer
**gets** the image first before it subsequently deletes the image later. My image does show up now. Nice.

But, it has a downside. A deal-breaking one. The show() method of PIL is for **debugging** only. Which means, the lib try to detect the OS
(Windows, Mac, Unix) and open the default image viewing program, so it doesn't have any method
to close the image. Cool. Now I have 2 choices. Either I implement a GUI (Graphical User Interface) or I find a way to
close that damned image. There's no no no way I'm learning PyQT or TkInter for a small program like this. So I blindly push forward on
the road that will lead me to hell. 

The Microsoft Photos program is the program showing my mirrored image. So I only have to figure out how to close a program that I knew the name of, right?
Not so fast. What if the user has several instances of Microsoft Photos opened around for other purposes? I can't afford to kill everything,
this would be a catatrosphe. I don't want to get a 1-star review IRL by my friend. Or how about mapping a certain hotkey to `ALT + F4` to close the image?
That would work all the time, and fail all the time too. In the rare event of the user opens the mirror, then focuses on another program,
I will be at fault for closing their working program with hours of work unsaved. Making sure to focus on the image viewer first?
Yeah I guess I could do that, but I'm not taking any chances. I will find a reliable way to do it.

From what I understand, my python script calls the image viewer, and the viewer is a subprocess of my script. There has to be a way
to close a process, or a subprocess. I **supposedly** have a handle called "Pillow" before. A command line to kill a process is enough?
` taskkill /FI "WindowTitle eq Pillow*" /T /F ` kills a program called "Pillow". Yup, doesn't work. No idea why. Moving on.
I look into psutil and subprocess libraries, which provides ways to open and kill any process. Here I learned about PID (Process ID)
and how to close a program with a certain PID.

```python
def kill_proc_tree(pid, including_parent=True):    
    parent = psutil.Process(pid)
    children = parent.children(recursive=True)
    for child in children:
        child.kill()
    gone, still_alive = psutil.wait_procs(children, timeout=5)
    if including_parent:
        parent.kill()
        parent.wait(5)
```

Kills a process and all it's subprocesses. Does. Not. Work. Zero idea why.

Instead of using os.system() command to open the image viewer, I switch to subprocess.Popen() to get a proper handle on the image viewer.
Long story short, that handle is also wrong. It doesn't point to the Microsoft Photos program. I give up for the mean time.

# 20/05/2020 The hell (literally, not an exclamation)

### Recommended music: Gián đoạn - Thành Luke

Why is life so hard on me? I just want to close a photo, please.
