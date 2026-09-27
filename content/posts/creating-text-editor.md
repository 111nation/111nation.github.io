+++
date = '2026-09-27T16:50:47+02:00'
draft = true
title = 'Editor - How I Made a Zero Dependency Text Editor Like Vim In C Using POSIX System Calls'
categories = ["Unix System Programming"]
tags = ["c", "systems programming", "unix", "tool"]
cover = "https://github.com/user-attachments/assets/f325c308-e128-4adf-a8c8-c9cbb00c8d5c"
+++
<br />

[Editor](https://github.com/111nation/Editor) is an extremely simple-to-use terminal text editor. It was written in C and it offers simplicity while being very responsive and performant. Editor was written using the [antirez's kilo guide.](https://viewsourcecode.org/snaptoken/kilo/index.html)

<video width="100%" controls autoplay loop muted src="https://github.com/user-attachments/assets/9c836c58-f7fe-4341-9045-596a26993495"></video>

Making Editor was a long but rewarding process. I wrote C for **the first time!** I've been avoiding doing it for a while (lol). I took the time to actually understand writing Editor while following antirez's kilo - Make Your Own Text Editor guide.

## Why did I do this?

As someone who is really into operating systems, I naturally picked up the holy grail operating system book, *'Operating Systems: Design and Implementation by Andrew Tanenbaum'*. This book walks you through how MINIX 3 or any UNIX/POSIX-compliant operating systems like Linux, macOS, FreeBSD, and formerly Windows work at a low level. 

My desire of actually making use of this information manifested in me making a very low level terminal utility. 


*\'But how does making text editor teach you about operating systems?\'* 


Suprisingly a lot! My goal with Editor was to not use any 3rd party libraries, only restricting myself to core system libraries and system calls. This forced me to see how userland utilities behave at the lowest level, inevitably uncovering the 'blackboxes' of how Operating Systems function.

## Getting Started



### Printing to the terminal

Before we are able to modify files and write to them we need to, well, figure out how we are going to display our text and statuses to the terminal. According to both the *Operating Systems* book and antirez's guide, terminals by default are loaded into **cooked** or **cononical mode**. This means that a shell such as `/bin/bash` allow you to type a full command such as

```bash
~ $ ls /home/chloe/ | grep 'my file'
```

and `/bin/bash` only receives those key inputs that represent a command when you press <kbd>Enter</kbd>. 

This is problematic, because we want Editor to see every keystroke without you having to press <kbd>Enter</kbd> every single time. What we need is to be in **raw** or **non-cononical mode**. This mode allows Editor to see key inputs the exact moment you press them.

To achieve this, we need to implement a few system calls in C which ask the operating system to switch terminal modes. In the code snippet below. We use [`tcgetattr`](https://man7.org/linux/man-pages/man3/tcgetattr.3.html) to get the terminal settings or *attributes*. It is extremely common to pass pointers to functions so that functions as `tcgetattr` are able to write to external data structures. 

We save these settings in `esettings.orig_termios` struct because we want to be able to restore these settings later. We then duplicate these settings and apply our settings using bitwise operations. To save time, this will be explained briefly, the [`termios`](https://man7.org/linux/man-pages/man3/termios.3.html) data structure holds binary integers that indicate: 

* `raw.c_iflag` - terminal input settings
* `raw.c_oflag` - terminal output flags
* `raw.c_cflag` - terminal control modes
* `raw.c_lflag` - terminal local modes

We turn off settings performing a bitwise **and**, `&=`, with the **not** of the bits that represent that setting, `~(BRKINT)`. We **or** multiple operations using a pipe symbol `|` to allow us to do this at once. There is an exception, with `raw_c_cflag` we want **add the settings** to indicate a character should be represented b 8 bits. That is why we **or** `CS8` into the settings, to put those settings bits there. 

> *If it isnt clear, antirez explains it very well, [check it out](https://viewsourcecode.org/snaptoken/kilo/02.enteringRawMode.html#:~:text=We%20can%20set,common%20in%20C.)*

With `raw.c_cc`, we specify timeout settings. The API is super clean and clear, it uses constants to indicate positions where we can specify specific time settings for `VTIME` and `VMIN`. I want to also write clean code like this!

We commit our changes to the terminal using [`tcsetattr`](https://man7.org/linux/man-pages/man3/tcgetattr.3.html).

```c
void enableRawMode() {
	// Get current terminal settings
	tcgetattr(STDIN_FILENO, &esettings.orig_termios);

	// Duplicate terminal settings
	struct termios raw = esettings.orig_termios;

	// Modify settings on the duplicate
	raw.c_iflag &= ~(BRKINT | ICRNL | INPCK | ISTRIP | IXON);
	raw.c_oflag &= ~(OPOST);
	raw.c_cflag |= (CS8);
	raw.c_lflag &= ~(ECHO | ICANON | ISIG | IEXTEN);

	raw.c_cc[VMIN] = 0; // read() can terminate with a min of 0 bytes recieved
	raw.c_cc[VTIME] = 1; // read() maximum wait time is 100 milliseconds

	// Commit to raw mode by passing the duplicate
	tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw);
}
```

Let's say the user is done editing their text and exits the editor. There is a problem! The operating system still leaves the terminal in raw mode, which is very bad for other processes (programs). We restore old settings in `disableRawMode` by giving [`tcsetattr`](https://man7.org/linux/man-pages/man3/tcgetattr.3.html) the old terminal settings, `esettings.orig_termios`.



