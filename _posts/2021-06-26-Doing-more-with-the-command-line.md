When I first tried to improve my terminal skills, I focused on learning as many Bash commands as I could, but just a few of them are far more helpful on a day-to-day basis than the rest. The following simple techniques boost efficiency by saving on unnecessary typing, and allow faster development without needing a mouse.

### 1. Quickly find previous commands

Using the terminal means issuing a lot of commands. These commands often require a complex set of options, which can be hard to remember. It would be very time-consuming to have to check the *man* pages or look at Google every time I want to use a command, especially when I know that I have used it before.

I could use the shortcut *Ctrl-r* for a reverse command search, and the closest match with previously entered commands would be shown to me as I type. However, this doesn't let me easily see all the variations and different options that I have used.

Alternatively, I can see everything that I have previously typed into the terminal with the *history* command. But then I need to manually scroll through the list of the past commands I have entered (on Debian and Ubuntu, the default is set to the last two thousand) for the one I want.

The best approach is to filter the history by removing unrelated lines. A popular way to do this is to pipe the output of *history* through to *grep*.

Let's say that last week I was working on a Python project, but I don't remember the exact options that I used with the test runner. I can easily search for matching commands in the history:

```bash
~$ history | grep "pytest"
 1010  pip install pylint pytest pytest-cov
 1322  pytest --version
 1324  pytest
 1426  pytest --cov=myproject
 1427  pytest -v
 1428  pytest --pdb
 1629  pytest --cov=myproject --pdb
 1630  pytest -x --pdb
```

Here is everything in the history that refers to *Pytest*: the command that installed it, and the commands that ran it with a variety of options. After looking at them, I decide I want the one that uses coverage and will activate the debugger if a test fails.

I want to run this command again, so (after making sure I am in the correct directory for my Python project) I just enter *!* followed by its line number: 

```bash
~$ !1629
pytest --cov=myproject --pdb
```

This is now the most recent command in the history, and as soon as I improve my Python code and want to test again, I can just hit the *up arrow* to repeat it.

### 2. Modify past commands with *fc*

Simply reusing commands from the history is sufficient a lot of the time. Even when I do want to modify a command, it is often one that I ran recently and the changes I need are fairly simple. This means that just hitting the *up arrow* a couple of times and making alterations on the command line (using the shortcuts discussed in point 7 below) will be enough.

Yet sometimes a command far back in the history is only a basis for what I want to do now: I may need to use a new option, or perhaps the filepath it should access is different. Here *fix command* is far more efficient than manually typing out a new line.

Let's say that I am starting a new project, and while I need to install a local version of *Pytest*, I also want to alter which additional packages I include. I can just type *fc* and the relevant line number:

```bash
fc 1010
```

The default editor (in my case, *Vim*) will open to let me modify the command that the number refers to:

```
pip install pylint pytest pytest-cov
```

I can now make whatever changes I want and save it for the modified command to run. If I quit the editor without saving, the original command will run instead.

### 3. Aliases

If you are frequently typing out long commands, it is helpful to use aliases. They customise your workflow and make powerful instructions possible with just a few keypresses.

They are also useful when you want to make sure that commands will always be called with specific options. A popular example is to make *rm* into an alias for *rm -i*. This has proved helpful for me (as it has for many people) when I have deleted several files, and then tried to move or copy another file, only to find that my fingers are on autopilot and have typed in *rm* again, permanently deleting a file that I had wanted to keep with no confirmation to make sure!

```bash
~$ alias rm="rm -i"
~$ touch unnecessary
~$ rm unnecessary
rm: remove regular empty file 'unnecessary'?
```

Similarly, while the earlier tip of searching for past commands with *history \| grep* is helpful, it becomes a strain to have to repeatedly type it out. A better solution would be to create an alias for it:

```bash
~$ alias hst="history | grep"
```

Now you only have to write `hst "pytest"`and the terminal will print the filtered history as if you had written the whole command.

Aliases are not persistent by default, and will be forgotten when you start a new session. To use aliases automatically in every session, add them to the `~/.bashrc` file. Either use an editor or just append to the end of the file like this:

```bash
~$ echo 'hst="history | grep"' >> ~/.bashrc
```

While there is also the option of using custom Bash functions for more complex logic, aliases are sufficient for the most common use cases.

### 4. Tab autocomplete

This one might seem a little obvious to mention, but I was doing a lot of work in the terminal before I started to properly use autocomplete.

For example, imagine that these files are in a directory:

`apples.py gamma.py gemini.js pears.js pears.test.js`

To access *apples.py*, I only need the first letter. By pressing *Tab*, the rest of the filepath gets filled in for me, just as it would in an IDE:

| keystroke | terminal |
| -------------- | ---------- |
|                    | ~$ cat        |
| a                 | ~$ cat a     |
| <Tab\> | ~$ cat apples.py |

The letter *g* could refer to two different files. Hitting *Tab* twice will show me both possible options:

| keystroke | terminal |
| -------------- | ---------- |
|                    | ~$ cat        |
| g                 | ~$ cat g     |
| <Tab\><Tab\> | ~$ cat g <br> gamma.js gemini.py |

Since only the first letter is the same, typing the second letter will clarify which file I want:

| keystroke | terminal |
| -------------- | ---------- |
| | ~$ cat g |
| e | ~$ cat ge |
| <Tab\> | ~$ cat gemini.js |

If there are multiple files with similar names, pressing *Tab* will also fill in the part common to both:

| keystroke | terminal |
| -------------- | ---------- |
|| ~$ cat |
| p | ~$ cat p |
| <Tab\> | ~$ cat pears |
| t | ~$ cat pears.t |
| <Tab\> | ~$ cat pears.test.js |

Autocomplete can also be used to start programs. For example, once I have the *intellij-idea-community* IDE installed, I can run it by just typing enough to uniquely identify the program (such as '*intell'*) and then hitting *Tab* and *Enter*.

A side effect of autocomplete on my workflow has been a tendency (within the constraints imposed by each project) to give files unique starting letters so that I can access them more easily. For instance, I would prefer to name my files *barIcon* and *chairIcon* rather than *iconOne* and *iconTwo*. Descriptive names are a better practice anyway, and I can now reach each file with a single *b* or *c*.

### 5. *Locate*, *find*, and recursive *grep*

One of the last things that was dragging me back to a file manager was needing to find something and not remembering its exact location. But there is no need to click the search button when it is so easy to directly use the underlying *locate* and *find* commands.

The main difference between them is that *locate* uses a pre-existing database, while *find* searches manually. If the database is up-to-date (the last update happened after the file you are looking for was created) then *locate* is definitely the faster and better option.

For example, this command will locate all Python files that end with the word 'schemas':

```bash
~$ locate "schemas.py"

/home/projects/flask_cinema_project/webapp/schemas.py
/home/tutorials/marshmallow_tutorial/practice_schemas.py
```

Filenames can be longer than the search text, which is why *practice_schemas.py* is included in the results, while *schemas_practice.py* would not be, because it doesn't contain an exact match for the search string. 

The *find* command is useful if you want to search by specific criteria, such as by user or access time. It is best to limit the scope as much as possible when you need to use it. This means being at the lowest point in the directory tree that you are certain is still above your file. Your search will then be localised, and will only need a small fraction of the time that it would take to go through everything on the system.

For instance, when looking for a JavaScript file that I wrote, I could start *find* from my projects directory, removing the need to check through the system files which obviously do not contain what I am looking for. If I could remember which project the file belongs to, then I could narrow things down even further by starting in that specific project's directory.

You can use globs or regular expressions to match multiple files. So within the current directory, I could find JavaScript test files that refer to 'Seat' like this:

```bash
~$ cd myproject

~/myproject$ find . -name "*Seat*test*.js"

./frontend/src/components/__tests__/SelectedSeats.test.js
./frontend/src/components/__tests__/SeatLayout.test.js
./frontend/src/components/__tests__/SeatIcon.test.js
./frontend/src/components/__tests__/SeatRow.test.js
```

To search for specific text inside files, use *grep* in recursive mode instead. For example, I could see which of my React components use *reduce* like this:

```bash
~/myproject$ grep -r "reduce" .

./frontend/src/components/SnackReserver.js:     .reduce((all, item) => ({ ...all, [item.id]: item.quantity }), {}),
./frontend/src/components/SnackInventory.js:    {snacks.reduce((total, item) => total + item.quantity * item.price, 0)}
```

This will look through all the files in the current directory and any subdirectories for the text you want.

### 6. The *tree* command

The main purpose of a graphical file manager is to visualise the directory layout, but even this can be handled equally well from the terminal. Just using *find* without any arguments will give a flat representation of the structure beneath the current directory:

```bash
~$ find

.
./outer
./outer/file_one.js
./outer/deeper
./outer/deeper/file_two.js
./outer/deeper/deepest
./outer/deeper/deepest/file_three.js
```

This is OK, but the output from the *tree* command is much clearer:

```bash
~$ tree

.
└── outer
    ├── deeper
    │   ├── deepest
    │   │   └── file_three.js
    │   └── file_two.js
    └── file_one.js

3 directories, 3 files
```

If the number of directories and files make the output unreadable, then just limit depth with the -L option:

```bash
~$ tree -L 2

.
└── outer
    ├── deeper
    └── file_one.js

2 directories, 1 file
```

*Tree* is not included as default but has to be installed (`apt get install tree` on Debian Linux). While it doesn't get mentioned as often as other commands like *less*, *cat* or *top*, I find that I use it more frequently, as I often want a quick overview of where everything is in my project. 

### 7. Learn the Bash shortcuts, or change to *Vi* mode

Many commands you type into the terminal are similar to each other. For instance, you may want to view the contents of a file, then edit it, and finally run it. Pressing the *up arrow* allows you to reuse the last command, which you can then modify as needed.

Knowing appropriate shortcuts for editing makes this process much faster. Bash has the *Emacs* shortcuts built in, and for example, you can use *Alt-b* to go back one word at a time, or *Ctrl-a* to go back to the start of the line.

This is great if you already know these shortcuts, but the terminal isn't fixed to the *Emacs* approach, and you don't necessarily have to use them. If you prefer, you can use the same key combinations that *Vi* uses instead. Just as it's easy to use *Vi* style key-bindings in an IDE through a plug-in for *Intellij* or *Visual Studio Code*, the terminal can be set to use them as well:

```bash
~$ set -o vi
```

(use `set -o emacs` to change it back)

Now you can type *Esc* to enter normal mode, *^* to go to the start of the line, and then *cw* to change the word, just like in *Vim*:

| keystrokes | terminal |
| -------------- | ---------- |
| <Esc\> | ~$ cat file.py |
| ^ | ~$ cat file.py |
| cw | ~$ file.py |
| python | ~$ python file.py |

Note that because it is *Vi* and not *Vim*, some combinations do not work. For instance, using *ciw* to *change-inner-word* doesn't work, while *cw* or *ce* does. This subset is still sufficient for most one line editing tasks.

The chief drawback, apart from having to know the *Vi* key-bindings, is that the current mode is not displayed like it would be in an actual *Vim* editor, so you have to mentally keep track of whether you are in insert or normal mode. This can take a little getting used to, but with practice and a few precautionary presses of the *Esc* key, it becomes a real time-saver.

Just as with aliases, add the *set* command to `~/.bashrc` to make it permanent.

### 8. Handling multiple files

A final tip to minimise unnecessary keystrokes is to apply a command to multiple files simultaneously.

If I want to create a series of blank files in the same directory, I only need to use one *touch* command:

```bash
~$ touch a b c
~$ tree
.
├── a
├── b
└── c

0 directories, 3 files
```

I can also move the files together to a different directory:

```bash
~$ mkdir letters
~$ mv a b c ./letters/
~$ tree
.
└── letters
    ├── a
    ├── b
    └── c

1 directory, 3 files
```

This simple usage doesn't work at a deeper level than the current directory, and trying to create files that way will end up with an odd structure:

```bash
~$ mkdir deeper
~$ touch ./deeper/d e f
~$ tree
.
├── deeper
│   └── d
├── e
└── f

1 directory, 3 files
```

Instead, I can just use braces without spaces:

```bash
~$ mkdir deeper
~$ touch ./deeper/{d,e,f}
~$ tree
.
└── deeper
    ├── d
    ├── e
    └── f

1 directory, 3 files
```

I can create multiple directories in the same way, and add a path to reach them with the -p option. This will create any necessary parent directories:

```bash
~$ mkdir -p numbers/{1,2,3}
~$ tree
.
└── numbers
    ├── 1
    ├── 2
    └── 3

4 directories, 0 files
```

Finally, I can use brace expansions to auto-generate filenames that follow a range:

```bash
~$ touch {1..4}_photo
~$ tree
.
├── 1_photo
├── 2_photo
├── 3_photo
└── 4_photo

0 directories, 4 files
```

These ranges can also be alphabetical:

```bash
~$ touch pic_{a..d}
~$ tree
.
├── pic_a
├── pic_b
├── pic_c
└── pic_d

0 directories, 4 files
```

This makes it much easier to create, rename, or move files in bulk without having to use a file manager.

### Summary

It obviously isn't necessary to avoid graphical tools in all circumstances, but many tasks can be handled more efficiently with just a terminal and a small amount of typing. I can easily see an overview of my project structure, find missing files, create nested directories, and repeat complex instructions without needing to use a mouse.

### Resources

There are many articles and tutorials covering more advanced usage of these eight points.

1. For detailed information on leveraging the Bash command history, see [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-use-bash-history-commands-and-expansions-on-a-linux-vps) and [this tutorial](https://www.howtogeek.com/howto/44997/how-to-use-bash-history-to-improve-your-command-line-productivity/).

2. The *fc* command has a lot of extra functionality. [Here is a good tutorial](https://shapeshed.com/unix-fc/).

   To set *Vim* as the default editor, add `export EDITOR=vim` to `~/.bashrc` (or substitute the editor you prefer to use).

3. A tutorial on creating Bash aliases is available [here](https://www.marksanborn.net/linux/creating-useful-bash-aliases/).

   A list of helpful aliases to create (including *history \| grep* and an alternative to *rm -i* that moves deleted files to the Trash instead) can be found [here](https://opensource.com/article/19/7/bash-aliases).

   See this guide for [appending text to the end of files](https://www.cyberciti.biz/faq/linux-append-text-to-end-of-file/).

   There is also the option of [Bash functions](https://ryanstutorials.net/bash-scripting-tutorial/bash-functions.php) for more complicated logic.

4. See [this guide](https://www.thegeekstuff.com/2013/12/bash-completion-complete/) for very advanced uses of Bash autocomplete.

5. Here are detailed tutorials for the [*find*](https://www.howtoforge.com/tutorial/linux-find-command/), [*locate*](https://www.howtoforge.com/linux-locate-command/), and [*grep*](https://tldp.org/LDP/GNU-Linux-Tools-Summary/html/x7969.htm) commands. All three have a lot of advanced options, which makes them even more useful in niche situations.

   See [this tutorial for using globs](https://linuxhint.com/bash_globbing_tutorial/) when searching for files.

6. Here is a guide to the standard [*Emacs* style terminal shortcuts](https://ostechnix.com/list-useful-bash-keyboard-shortcuts/).

   Here is a list of [*Vi* key bindings](https://hea-www.harvard.edu/~fine/Tech/vi.html).

   A comprehensive guide to using *Vim*:

   [Drew, Neil. *Practical Vim*, 2nd ed. The Pragmatic Bookshelf, 2015.](
https://pragprog.com/titles/dnvim2/practical-vim-second-edition/)

7. Here is a complete list of [*tree* command options](https://linux.die.net/man/1/tree) from the official documentation.

   See [here](https://vitux.com/linux-tree-command/) and [here](https://www.tecmint.com/linux-tree-command-examples/) for two blog posts covering more advanced usage of the *tree* command.

8. For details on *mkdir*, including the parents option, see [this tutorial](https://www.howtoforge.com/linux-mkdir-command/).

   Three tutorials on using brace expansions and ranges in Bash are available [here](
https://wiki.bash-hackers.org/syntax/expansion/brace), [here](https://www.linuxjournal.com/content/bash-brace-expansion), and [here](https://linuxhint.com/bash_brace_expansion/).
