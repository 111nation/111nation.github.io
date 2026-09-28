+++
date = '2026-09-27T16:50:47+02:00'
draft = true
title = 'Editor - How I Made a Zero Dependency Text Editor Like Vim In C Using POSIX System Calls'
categories = ["Unix System Programming"]
tags = ["c", "systems programming", "unix", "tool"]
cover = "https://github.com/user-attachments/assets/f325c308-e128-4adf-a8c8-c9cbb00c8d5c"
+++
<br />

[Editor](https://github.com/111nation/Editor) is an extremely simple-to-use terminal text editor. It was written in C and it offers simplicity while being very responsive and performant. Editor was written using the [antirez's Kilo guide.](https://viewsourcecode.org/snaptoken/kilo/index.html)

<video width="100%" controls autoplay loop muted src="https://github.com/user-attachments/assets/9c836c58-f7fe-4341-9045-596a26993495"></video>

Making Editor was a long but rewarding process. I wrote C for **the first time!** I come from a Rust and C++ background but the more I delve into systems programming, the more I realize the need for understanding C as well as how things work at a low-level.

## Why did I do this?

As someone who is really into operating systems, I naturally picked up the holy grail operating system book, *'Operating Systems: Design and Implementation by Andrew Tanenbaum'*. This book walks you through how MINIX 3 or any UNIX/POSIX-compliant operating systems (like Linux, macOS, FreeBSD) as well as formerly Windows NT work at a low-level. 

I desired to use the valuable knowledge from this book, which manifested in building a terminal utility.

*'But how does making text editor teach you about operating systems?'* 


Suprisingly, a lot! My goal with Editor was to not use any 3rd party libraries, only restricting myself to core system libraries and system calls. This forced me to see how userland utilities behave at the lowest level, inevitably uncovering the 'blackboxes' of how Operating Systems function.

## Getting Started

{{< alert type="warning" title="Warning" >}}
Code examples are purposefully simplified for illustrative purposes, view the whole source code [here](https://github.com/111nation/Editor)

It is impossible for me to go through everything in this blog, so the following will be a heavily stripped down to the bare essentials that relate to systems programming and core features.
{{</ alert >}}

### Setting up the terminal 

Before we are able to modify files and write to them we need to, well, figure out how we are going to display our text and statuses to the terminal. According to both the *Operating Systems* book and antirez's guide, terminals by default are loaded into **cooked** or **cononical mode**. This means that a shell such as `/bin/bash` allow you to type a full command such as

```bash
~ $ ls /home/chloe/ | grep 'my file'
```

and `/bin/bash` only receives those key inputs that represent a command when you press <kbd>Enter</kbd>. 

This is problematic, because we want Editor to see every keystroke without you having to press <kbd>Enter</kbd> every single time. What we need is to be in **raw** or **non-cononical mode**. This mode allows Editor to see key inputs the exact moment you press them.

To achieve this, we need to implement a few system calls in C which ask the operating system to switch terminal modes. In the code snippet below. We use [`tcgetattr()`](https://man7.org/linux/man-pages/man3/tcgetattr.3.html) to get the terminal settings or *attributes*. It is extremely common to pass pointers to functions so that functions as `tcgetattr()` are able to write to external data structures. 

We save these settings in `esettings.orig_termios` struct because we want to be able to restore these settings later. We then duplicate these settings and apply our settings using bitwise operations. To save time, this will be explained briefly, the [`termios`](https://man7.org/linux/man-pages/man3/termios.3.html) data structure holds binary integers that indicate: 

* `raw.c_iflag` - terminal input settings
* `raw.c_oflag` - terminal output flags
* `raw.c_cflag` - terminal control modes
* `raw.c_lflag` - terminal local modes

We turn off specific settings by performing a bitwise **AND** (`&=`) with the **NOT** (`~`) of those flags, such as `~(BRKINT)`. We can combine multiple flags using the pipe symbo ('|') to clear them all at once. 

There exception here is `raw.c_cflag`, we want to **turn on** a settings to specify that characters should be 8 bits long. To do this, we use a bitwise **OR** (`|=`) to force the `CS8` bits to a value of `1`. 

{{< alert type="info" title="Tip" >}}
> *If it isnt clear, antirez explains it very well, [check it out](https://viewsourcecode.org/snaptoken/kilo/02.enteringRawMode.html#:~:text=We%20can%20set,common%20in%20C.)*
{{</ alert >}}

With `raw.c_cc`, we specify timeout settings. The API is super clean and clear, it uses constants to indicate positions where we can specify specific time settings for `VTIME` and `VMIN`. 

We commit our changes to the terminal using [`tcsetattr()`](https://man7.org/linux/man-pages/man3/tcgetattr.3.html).

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

Let's say the user is done editing their text and exits the editor. There is a problem! The operating system still leaves the terminal in raw mode, which is very bad for other processes (programs). We restore old settings in `disableRawMode()` by giving [`tcsetattr()`](https://man7.org/linux/man-pages/man3/tcgetattr.3.html) the old terminal settings, `esettings.orig_termios`.

Just like that we have successfully prepped the terminal for Editor :)

### Key Input, The Right Way


*'I thought we were done with key input!' :(*

Turns out, even though Editor recieves key input instantly, this is being received as byte streams. It is our job to format and interpret these bytes.

The way this is done is by defining a enum that accepts all valid key input for editor. Due to the nature of the enum extending past	`128` for special non-ascii characters, we use an `int` data type instead of a `char` to prevent integer overflow.

```c
enum editorKey {
	BACKSPACE = 127,
	ARROW_LEFT = 1000,
	ARROW_UP,
	ARROW_DOWN,
	ARROW_RIGHT,
	DEL_KEY,
	PAGE_UP,
	PAGE_DOWN,
	HOME_KEY, 
	END_KEY,
	NO_KEY_PRESS,
};
```

We then wrap all key input logic, which maps ascii and special character streams into a single integer to process easily, as `get_key()`. When we are in raw mode, we receive the key input immediately, if a user enters a regular letter (capital letter or small letter), we receive that and return it as its regular ascii code. While special keys like arrow keys, <kbd>Home</kbd> and <kbd>Delete</kbd> keys arent usually sent as standard ascii, rather as [ansi escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code). 

The [`read`](https://man7.org/linux/man-pages/man1/read.1p.html) system call is used to read from a file. Since the nature of UNIX operating system treating everything as a file, we can read the contents of `STDIN_FILENO` which is a standardized file descriptior (file pointer) to the input buffer connected to the current terminal session.

[VT100](https://espterm.github.io/docs/VT100%20escape%20codes.html) escape codes start with an escape sequence, `\x1b` (hex) or `\033` (oct), and follow a stream of ascii characters to indicate a special character or special terminal operation. For example, right arrow key is represented as `\x1b[C` or your <kbd>Delete</kbd> key as `\x1b3~`. 

Certian keys like <kbd>Enter</kbd> and <kbd>Backspace</kbd> do not need to be processed using the enum. These keys are already represented as `\r` and `\b` using pure ascii.

{{< alert type="info" title="Note" >}}
> *Our goal with `get_key()` is to represent a single key press as a single integer. This makes life much more easier for processing keys!*
{{</ alert >}}

```c
int get_key() {
	char c = '\0';
	ssize_t res = read(STDIN_FILENO, &c, 1);

	// If not special character, return ascii value as is
	if (c != '\x1b') return c;

	// Capture special keys, i.e Arrow keys
	char special[3];
	
	// Special characters have trailing characters left to read
	read(STDIN_FILENO, &special[0], 1);
	read(STDIN_FILENO, &special[1], 1);

	if (special[0] == '[') {
		switch (special[1]) {
			case 'A': return ARROW_UP;
			case 'B': return ARROW_DOWN;
			case 'C': return ARROW_RIGHT;
			case 'D': return ARROW_LEFT;
			case 'H': return HOME_KEY;
			case 'F': return END_KEY;
		}

		// Numbers as the 2nd char indicates more special keys
		if (special[1] >= '0' && special[1] <= '9'){
			read(STDIN_FILENO, &special[2], 1) != 1;

			if (special[2] != '~') return '\x1b';
			switch (special[1]) {
				case '1': case '7': return HOME_KEY;
				case '4': case '8': return END_KEY;
				case '3': return DEL_KEY;
				case '5': return PAGE_UP;
				case '6': return PAGE_DOWN;
			}
		} 
	}

	return '\x1b'; // We got an escape code but it was incomplete
}
```

After representing complex keys as a single integer, we can pass this to `process_keypress()` which makes editor do something. It is always excellent practice to seperate programs into modules with specific purposes!

## Displaying Logic

### Structuring Text

To be able to manipulate and display text nicely we need to structure text into an easily iterable structure. To achieve this, Editor uses a main struct under the global editor settings, `esettings`, struct.

As seen below, the text is structured so we can traverse easily. `erow` is a data type struct that stores data pertaining to a single row. If you notice, each row stores actual ascii characters, `char *chars`. `char *render` is similar with the exception of displaying tabs as four spaces (defined by `TAB_STOP` preprocessor) instead of a regular ascii `\t`. This is done because displaying a `\t` registers as a single character to our cursor positioning system `cx` and `cy`, but a tab actually occupies more than one character visually on your screen! This prevents cursor movement bugs when a row's `indent_level` is greater than 0.

We store all the rows as an array of `erow` struct. 

{{< alert type="info" title="Note" >}}
> *By doing this, we made it easy for our cursor to jump between rows and easily query information of a row without full **O(1)** operations on the whole file contents.*
{{</ alert >}}

```c
#define TAB_STOP 4

typedef struct erow { // Single row data
	int index;
	int size;		  // char size
	int indent_level;
	int rsize;		  // render size
	char *chars;	
	char *render;
	...
} erow;

struct editorConfig {
	...
	unsigned int cx, cy; 			// Cursor position in file
	unsigned int rx; 				// Rendered cursor position in rendered file
	unsigned int ws_col, ws_row;	// Screen dimensions
	int numrows;					// Number of rows in file
	int rowoff; 					// Line number displayed as first line on screen 
	int coloff;						// Character of current line displayed as first character on screen
	erow* row;						// File row data
	...
};
```

### Getting Window Size
Before we can start displaying text to the terminal, we need to query the operating system for the terminal window dimensions. A self documenting function, `get_window_size()` encapsulates this function for us.

Due to the nature of a text-based terminal, the window dimensions will be amount of character rows by character columns. An extremely important POSIX system call, [`ioctl()`](https://man7.org/linux/man-pages/man2/ioctl.2.html) is used to query the terminal driver for the terminal dimensions.

`ioctl()` is used for interacting with special files that [`read`](https://man7.org/linux/man-pages/man3/read.3p.html) or [`write()`](https://man7.org/linux/man-pages/man2/write.2.html) cannot interact with. This are typically drivers for example.  We inform `ioctl()` that we are querying for the output stream of the terminal (`STDOUT_FILENO`) and we would love to query the window size (`TIOCGWINSZ`). We pass a [`winsize`](https://man7.org/linux/man-pages/man2/TIOCSWINSZ.2const.html) struct into `ioctl()` which will be modified with the terminal window size.

If this is unsuccesful, we opt by using `write()` system call to print special [VT100](https://espterm.github.io/docs/VT100%20escape%20codes.html) escape codes that move the cursor to the far ends of the window and get the relative cursor position.

```c
int get_window_size(unsigned int *rows, unsigned int *cols) {
	struct winsize ws;

	if (ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws) == -1 || ws.ws_col == 0) {
		// Driver Query Unsuccessful
		if (write(STDOUT_FILENO, "\x1b[999C\x1b[999B", 12) != 12) return -1;
		return get_cursor_position(rows, cols);
	}

	*cols = ws.ws_col;
	*rows = ws.ws_row;
	return 0;
}
```
### Status Bar and Prompter

<img width="100%" alt="Screenshot_20260829_185529" src="https://github.com/user-attachments/assets/0fad41bc-162f-43d2-b12b-e21c19744b9e" />

Displaying text is relatively simple once we laid the foundation, to save on time, it will not be discussed thouroughly. All that is needed is to use the [`write()`](https://man7.org/linux/man-pages/man2/write.2.html) system call to the standard output stream (`STDOUT_FILENO`) while iterating through every row, `*esettings.row` to print every [**rendered characters**](/posts/creating-text-editor/#status-bar-and-prompter:~:text=As%20seen%20below,greater%20than%200.) `*esettings.row->render` to the terminal.

{{< alert type="info" title="Extra" >}}
> *Editor tracks which line and character appears first in view in the window via `esettings.rowoff` and `esettings.coloff`. Using this information we can modify which lines and characters appear first to allow scrolling. This is what `escroll()` does*
{{</ alert >}}

A function, `edraw_status()` is used to draw the white background status bar. We simply use `write()` into `STDOUT_FILENO` in order to write escape codes to move the cursor to the desired position and use `\x1b[7m` to change the color of the background.

<img width="100%" alt="Screenshot_20260829_185529" src="https://github.com/user-attachments/assets/f325c308-e128-4adf-a8c8-c9cbb00c8d5c" />

`prompt()` handles displaying and managing the text editor prompt. A prompt is used for a search function or indicating the name of a file you want to save as. Drawing the prompt takes a similar approach of moving the cursor one line below after printing the status message. Thereafter, an infinite loop polls for user input until a user enters text or cancels their prompt. 


{{< alert type="info" title="Extra" >}}
> *`prompt()` accepts a callback which allows other modules of the editor to be notified of every key input. This is how Editor supports incremental searching!*
{{</ alert >}}

## Implementing Text Editing

### File Operations & POSIX I/O

Opening a file is straightfoward, even with POSIX system calls, but string ownership requires careful memory management!

Before assigning `esettings.filename`, we must free any existing buffer pointer **deep copy** the incoming `filename` using [`strdup()`](https://man7.org/linux/man-pages/man3/strdup.3.html). Since file paths passed from stack frames or function arguments may not outlive the main program loop, creating a heap-allocated **deep copy** prevents dangling pointers and segmentation faults in future!

`eopen()` grabs each line of the file using [`getline()`](https://man7.org/linux/man-pages/man3/getline.3.html). Each line is stripped of any trailing line breaks (`\r`, `\n`) and passes the string representation of a row into `insert_row()` to construct our `erow` data structure. Finally, we reset `esettings.dirty = 0` to indicate the file hasn't been modified, thus the file is still clean :)

```c
void eopen(char* filename) {
	free(esettings.filename); // Clear previous file pointer
	esettings.filename = strdup(filename);
	
	eselect_syntax_highlight();

	FILE *fp = fopen(filename, "r");

	char *line = NULL;
	size_t linecap = 0;
	ssize_t len;
	while((len = getline(&line, &linecap, fp)) != -1) {
		// Remove Trailing '\n' and '\r'
		while (len > 0 && (line[len-1] == '\n' || line[len-1] == '\r')) len--;
		insert_row(esettings.numrows, line, len);
	}

	free(line);
	fclose(fp);

	esettings.dirty = 0;
}
```

To save a file, we prompt the user with `prompt()` for a file name if the user did not launch editor with an argument specifing the file to edit. We use [`open()`](https://man7.org/linux/man-pages/man2/open.2.html) to retrieve a file descriptor (file pointer) to the file. By specifying bit flags `O_RDWR | O_CREAT` we will open the file in read-write mode and if it does not exist already, the file will be created. `0644` indicates in octal the file permisions of the file. The owner, user who created the file, will have read and write permissions while everyone on the system will have read access.

We use `goto` statements to make error handling clean and free of duplicated clean up code. This prevents deep nested if statements and keeps the happy path at a low-indent level. Always strive for easily readable code!

We use [`ftruncate()`](https://man7.org/linux/man-pages/man3/ftruncate.3p.html) to resize the file to fit the contents of the user's text without wasting extra storage.

```c
void esave() {
	esettings.filename = prompt("Save file as:\t %s", NULL);

	int len;
	char *buf = erows_to_string(&len);

	int fd = open(esettings.filename, O_RDWR | O_CREAT, 0644);

	if (fd == -1) {
		goto cleanup_buf;
	}

	if (ftruncate(fd, len) == -1) {
		goto cleanup_fd;
	}

	if (write(fd, buf, len) != len) {
		goto cleanup_fd;
	}

	eset_message("%d bytes written to disk", len);

	close(fd);
	free(buf);
	esettings.dirty = 0;
	return;

	cleanup_fd:
		close(fd);
	cleanup_buf:
		free(buf);
	
	eset_message("Error Saving! I/O error: %s", strerror(errno));
}
```

## Conclusion

That covers all the core elements of Editor. I could not unfortunately explain a 1,000+ line codebase in one blog. Details regarding syntax highlighting and searching have been intentionally left out, but if you are interested in the project, view it on my [GitHub](https://github.com/111nation/Editor).
