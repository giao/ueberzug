# Überzug

Überzug is a command line util
which allows to draw images on terminals by using child windows.

Advantages to w3mimgdisplay:
- no race conditions as a new window is created to display images
- expose events will be processed,  
  so images will be redrawn on switch workspaces
- tmux support
- terminals without the WINDOWID environment variable are supported
- chars are used as position - and size unit
- no memory leak (/ unlimited cache)

## Overview

- [Installation](#installation)
- [Communication](#communication)  
  * [Command formats](#command-formats)
  * [Actions](#actions)
    + [Add](#add)
    + [Remove](#remove)
  * [Libraries](#libraries)
    + [Bash](#bash)
    + [Python](#python)
  * [Examples](#examples)

## Installation

```
$ sudo pip3 install ueberzug
```

## Communication

The communication is realised via stdin.  
A command is a request to execute a specific action with the passed arguments.  
(Therefore a command has to contain a key value pair "action": action_name)  
Commands are separated with a line break.

### Command formats

- json: Command as json object
- simple: Key-value pairs seperated by a tab,  
          pairs are also seperated by a tab  
- bash: dump of an associative array (`declare -p variable_name`)

### Actions

#### Add

Name: add  
Description:  
Adds an image to the screen.  
If there's already an image with the same identifier  
it will be replaced.

| Key           | Type         | Description                                                        | Optional |
|---------------|--------------|--------------------------------------------------------------------|----------|
| identifier    | String       | a freely choosen identifier of the image                           | No       |
| x             | Integer      | x position                                                         | No       |
| y             | Integer      | y position                                                         | No       |
| path          | String       | path to the image                                                  | No       |
| width         | Integer      | desired width; original width will be used if not set              | Yes      |
| height        | Integer      | desired height; original width will be used if not set             | Yes      |
| max_width     | Integer      | image will be resized (while keeping it's aspect ratio) if it's width is bigger than max width | Yes |
| max_height    | Integer      | image will be resized (while keeping it's aspect ratio) if it's height is bigger than max height | Yes |
| draw          | Boolean      | redraw window after adding the image, default True                 | Yes      |
| synchronously_draw | Boolean | redraw window immediately                                          | Yes      |

#### Remove

Name: remove  
Description:  
Removes an image from the screen.  

| Key           | Type         | Description                                                        | Optional |
|---------------|--------------|--------------------------------------------------------------------|----------|
| identifier    | String       | a previously used identifier                                       | No       |
| draw          | Boolean      | redraw window after removing the image, default True               | Yes      |


### Libraries

Just a reminder: This is a GPLv3 licensed project, so if you use any of these libraries you also need to license it with a GPLv3 compatible license.

#### Bash

First of all the library doesn't follow the posix standard,  
so you can't use it in any other shell than bash.  

Executing `ueberzug library` will print the path to the library to stdout.  
```bash
source "`ueberzug library`"
```

**Functions**:

- `ImageLayer` starts the ueberzug process and uses bashs associative arrays to transfer commands.  
- Also there will be a function named `ImageLayer::{action_name}` declared for each action.  
  Each of this function takes the key values pairs of the respective action as arguments.  
  Every argument of these functions has to be an associative key value pair.  
  `ImageLayer::{action_name} [{key0}]="{value0}" [{key1}]="{value1}" ...`  
  Executing such a function builds the desired command string according to the passed arguments and prints it to stdout.  

#### Python

First of all everything which isn't mentioned here isn't safe to use and  
won't necessarily shipped with new coming versions.  

The library is included in ueberzug's package.  
```python
import ueberzug.lib.v0 as ueberzug
```

**Classes**:

1. Visibility:  
   An enum which contains the visibility states of a placement.  
   
   - VISIBLE
   - INVISIBLE
2. Placement:  
   A placement to put images on.  
   
   Every key value pair of the add action is an attribute (except identifier).  
   Changing one of it will lead to building and transmitting an add command *if the placement is visible*.  
   The identifier key value pair is implemented as a property and not changeable.  
   
   Constructor:  
   
   | Name          | Type         | Optional | Description                                    |
   |---------------|--------------|----------|------------------------------------------------|
   | canvas        | Canvas       | No       | the canvas where images should be drawn on     |
   | identifier    | String       | No       | a unique string used to address this placement |
   | visibility    | Visibility   | Yes      | the initial visibility state<br>(if it's VISIBLE every attribute without a default value needs to be set) |
   | \*\*kwargs    | dict         | Yes      | key value pairs of the add action              |
   
   Properties:  
   
   | Name          | Type         | Setter | Description                          |
   |---------------|--------------|--------|--------------------------------------|
   | identifier    | String       | No     | the identifier of this placement     |
   | canvas        | Canvas       | No     | the canvas this placement belongs to |
   | visibility    | Visibility   | Yes    | the visibility state of this placement<br>- setting it to VISIBLE leads to the transmission of an add command<br>- setting it to INVISIBLE leads to the transmission of a remove command |
   
   **Warning**:  
   The transmission of a command can lead to an IOError.  
   (A transmission happens on assign a new value to an attribute of a visible Placement.  
   The transmission is delayed till leaving a with-statement if lazy_drawing is used.)
3. Canvas:  
   Should either be used with a with-statement or with a decorated function.  
   (Starts and stops the ueberzug process)

   Constructor:  
   
   | Name          | Type         | default  | Description                                    |
   |---------------|--------------|----------|------------------------------------------------|
   | debug         | bool         | False    | suppresses printing stderr if it's false       |
   
   Methods:  
   
   | Name             | Returns      | Description                          |
   |------------------|--------------|--------------------------------------|
   | create_placement | Placement    | prevents the use of the same identifier multiple times,<br>takes the same arguments as the Placement constructor (excluding canvas parameter) |
   | \_\_call\_\_     | Function     | Decorator which returns a function which calls the decorated function with the keyword parameter canvas=this_canvas_object.<br>Of course other arguments are also passed through. |
   
   Properties:  
   
   | Name          | Type                    | Setter | Description                          |
   |---------------|-------------------------|--------|--------------------------------------|
   | lazy_drawing  | context manager factory | No     | prevents the transmission of commands till the with-statement was left<br>`with canvas.lazy_drawing: pass`|
   | synchronous_lazy_drawing  | context manager factory | No     | Does the same as lazy_drawing. Additionally forces the redrawing of the windows to happen immediately. |




### Examples

Command formats:

- Json command format: `{"action": "add", "x": 0, "y": 0, "path": "/some/path/some_image.jpg"}`  
- Simple command format: `action add x   0   y   0   path    /some/path/some_image.jpg`  
- Bash command format: `declare -A command=([path]="/some/path/some_image.jpg" [action]="add" [x]="0" [y]="0" )`  

Bash library:

```bash
source "`ueberzug library`"

# process substitution example:
ImageLayer 0< <(
    ImageLayer::add [identifier]="example0" [x]="0" [y]="0" [path]="/some/path/some_image0.jpg"
    ImageLayer::add [identifier]="example1" [x]="0" [y]="0" [path]="/some/path/some_image1.jpg"
    read
    ImageLayer::remove [identifier]="example0"
    read
)

# group commands example:
{
    ImageLayer::add [identifier]="example0" [x]="0" [y]="0" [path]="/some/path/some_image0.jpg"
    ImageLayer::add [identifier]="example1" [x]="0" [y]="0" [path]="/some/path/some_image1.jpg"
    read
    ImageLayer::remove [identifier]="example0"
    read
} | ImageLayer
```

Python library:  

- curses (based on https://docs.python.org/3/howto/curses.html#user-input):  
```python
  import curses
  import time
  from curses.textpad import Textbox, rectangle
  import ueberzug.lib.v0 as ueberzug
  
  
  @ueberzug.Canvas()
  def main(stdscr, canvas):
      demo = canvas.create_placement('demo', x=10, y=0)
      stdscr.addstr(0, 0, "Enter IM message: (hit Ctrl-G to send)")
  
      editwin = curses.newwin(5, 30, 3, 1)
      rectangle(stdscr, 2, 0, 2+5+1, 2+30+1)
      stdscr.refresh()
  
      box = Textbox(editwin)
  
      # Let the user edit until Ctrl-G is struck.
      box.edit()
  
      # Get resulting contents
      message = box.gather()
      demo.path = ''.join(message.split())
      demo.visibility = ueberzug.Visibility.VISIBLE
      time.sleep(2)
  
  
  if __name__ == '__main__':
      curses.wrapper(main)
  ```
  
- general example:  
  ```python
  import ueberzug.lib.v0 as ueberzug
  import time
  
  if __name__ == '__main__':
      with ueberzug.Canvas() as c:
          paths = ['/some/path/some_image.png', '/some/path/another_image.png']
          demo = c.create_placement('demo', x=0, y=0)
          demo.path = paths[0]
          demo.visibility = ueberzug.Visibility.VISIBLE
  
          for i in range(30):
              with c.lazy_drawing:
                  demo.x = i * 3
                  demo.y = i * 3
                  demo.path = paths[i % 2]
              time.sleep(1/30)
  
          time.sleep(2)
  ```

Scripts:

- Mastodon viewer: https://github.com/seebye/ueberzug/blob/master/examples/mastodon.sh
- **F**zf **M**pd **U**ser **I**nterface: https://github.com/seebye/fmui
