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

We then wrap all key input logic, which maps ascii and special character streams into a single integer to process easily, as `get_key`. When we are in raw mode, we receive the key input immediately, if a user enters a regular letter (capital letter or small letter), we receive that and return it as its regular ascii code. While special keys like arrow keys, home and delete keys arent usually sent as standard ascii, rather as [ansi escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code). 

[VT100](https://espterm.github.io/docs/VT100%20escape%20codes.html) escape codes start with an escape sequence, `\x1b` (hex) or `\033` (oct), and follow a stream of ascii characters to indicate a special character or special terminal operation. For example, right arrow key is represented as `\x1b[C` or your delete key as `\x1b3~`. 

Certian keys like <kbd>Enter</kbd> and <kbd>Backspace</kbd> do not need to be processed using the enum. These keys are already represented as `\r` and `\b` using pure ascii, so we are able to fit this operating using one integer variable, which was not possible with other key input!

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
			...
			case 'H': return HOME_KEY;
		}

		// Numbers as the 2nd char indicates more special keys
		if (special[1] >= '0' && special[1] <= '9'){
			read(STDIN_FILENO, &special[2], 1) != 1);

			if (special[2] != '~') return '\x1b';
			switch (special[1]) {
				case '1': case '7': return HOME_KEY;
				...
				case '3': return DEL_KEY;
			}
		} 
	}

	return '\x1b'; // We got an escape code but it was incomplete
}
```

After representing complex keys as a single integer, we can pass this to `process_keypress` which makes editor do something. It is always excellent practice to seperate programs into modules with specific purposes!

## Implementing Text Editing

### Displaying Structured Text

To be able to manipulate and display text nicely we need to structure text into an easily iterable structure. To achieve this, Editor uses a main struct under the global editor settings, `esettings`, struct.

As seen below, the text is structured so we can traverse easily. `erow` is a data type struct that stores data pertaining to a single row. If you notice, each row stores their actual ascii `chars` which is/ to be seen in the file. While `render` is  the same characters, but instead of displaying tabs as a regular `\t` code, we perform tab stopping and display it as four spaces. This is done because displaying a `\t` registers as a single character to the cursor, but actually takes up more than one character on your screen! 

We store all the rows as an array of `erow` struct. 

```c
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

### Displaying Text



### File Operations

Opening a file is pretty easy even with POSIX system calls, there are few things to remember though!

We copy the name of the file from the function argument into `esettings.filename` using [`strdup`](https://man7.org/linux/man-pages/man3/strdup.3.html). Primitive C strings in runtime are stored as contiguous element, we cannot simply set `esettings.filename = filename`. This will produce what is called a **shallow copy**. A shallow copy only copies a pointer that refers to the same block of memory, so both `esettings.filename` and `filename` will incorrectly point to the same string! `filename` may be a literal or stack initiated variable which does not survive throughout the program execution. Thus makes `esettings.filename` risk pointing to a invalid address producing segmentation faults!


`eopen` grabs each line of a file and passes the string representation of a row into `insert_row` which structures the row properly into a proper `erow` format. `esettings.dirty` is merely there to track how many bytes have been modified compared to the original file. Since nothing was modified, the file is still clean :)

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

To save to a file we need a way to prompt the user for a file name


```c
void esave() {
	esettings.filename = prompt("Save file as:\t %s", NULL);
		if (esettings.filename == NULL) {
			eset_message("Save aborted");
			return;	
		}
		eselect_syntax_highlight();
	}

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
