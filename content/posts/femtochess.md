+++
title = 'Femtochess'
date = 2024-06-14T11:54:45-04:00
draft = true

description = 'Attempting to make a chess engine in 300 lines of C or less.'
type = "post"
showTableOfContents = true
tags = ["chess", "C", "tech"]

+++
## Femtochess -- My Attempt at Tiny Chess Engines
I've been intrigued by chess programming for as long as I've been programming. I've started more chess engine projects than I'd care to admit, but I have yet to
make a chess engine that actually works and plays by all the rules (castling, en passant, etc.).

Through all of my attempts, I have read a great deal about chess programming and watched a good amount of youtube videos. I have a really good idea of how to make
a chess engine run, I just haven't had the motivation to finish one in its entirety. That changes now. I figure if I blog along as I develop a chess engine, I have
a good chance of actually completing it in a reasonable time frame.

I'm going to call it Femtochess after a chess engine that I keep coming back to: [Nanochess by Oscar Toledo G.](https://nanochess.org/chess3.html). I know there
are tricks to the demoscene and keeping your source code tiny, but I can't fathom how you can have a fully working chess engine in only 1257 (nonblank) characters.
Someone also ported it to JavaScript and made a neat program with cool graphics called The Kilobyte's Gambit you can play against in browser
[here](https://vole.wtf/kilobytes-gambit/).

To make this more interesting for myself, I am giving myself the restriction of keeping the source code under 300 lines. This seems like a tough but achievable
goal for myself, and will force me to do only what is essential while making creative choices along the way. I might get the opportunity to abuse macros too!
Before I get started, I ought to explain my current view on how a chess engine works, more or less.

## Chess Engines
The way I see chess engines, there are three parts: the board representation, the move generator, and the evaluation/search.

The board representation is just the data structure/s you use to store the current state of the chess board and game. This isn't just where the pieces are, but
also whose turn it is, if there are any squares available for en passant capture, you could even store information about which pieces are pinned in the board
representation.

The move generator takes a current board state and spits out all the possible moves that could be made. This is where storing the extra information like which
pieces are pinned comes in handy, as you can skip generating those moves all together rather than generate the move then check if it's legal or not.

The evaluation assigns a score to a certain board state. You could imagine you can create a tree of board states, go to a certain depth, and evaluate all of the board states along the way and just traverse down the path that maximizes your score and minimizes your opponents. That is the heart of a chess engine. You can
also imagine how many leaf nodes that tree has even after a couple of levels, which is why chess engines are tricky to program.

## Getting Started
To get started, I'm going to make a new directory and set up a hello world with a makefile.

```makefile
# Makefile
CC = gcc
CFLAGS = -Wall -Wextra -O2

TARGET = femtochess

SRCS = femtochess.c

all: $(TARGET) count

$(TARGET):
	rm -f $(TARGET)
	$(CC) $(CFLAGS) -o $(TARGET) $(SRCS)

clean:
	rm -f $(TARGET)

count:
	@LINES=$$(grep -v '^\s*$$' $(SRCS) | grep -v '^\s*//' | grep -v '^\s*/\*' | grep -v '^\s*\*' | wc -l); \
	echo "$$LINES lines counted in $(SRCS)"


.PHONY: all clean
```

```c
// femtochess.c
#include "stdio.h"

int main()
{
        printf("Hello, world!\n");
        return 0;
}
```

The `count` target in the makefile is a little helper I put together to, well, count the number of lines of code (excluding whitespace and comments). So when
I make the project, I get this:

```bash
❯ make
rm -f femtochess
gcc -Wall -Wextra -O2 -o femtochess femtochess.c
6 lines counted in femtochess.c
❯ ./femtochess
Hello, world!
```

I think that's basically all that I need in the way of a build pipeline/dev environment. Now it's time to think about what the most compact yet simple to work
with board representation is.

### Board Representation
There are a couple of different ways to represent the board. The strongest chess engines use bitboards, which is using a 64 bit number where each bit represents
a square on the board. One of the numbers could represent everywhere on the board there is a white pawn, or everywhere on the board that you can attack with en
passant, for example. That is going to be outside the scope of this engine, and instead we will use the most basic and intuitive representation of a board, a
simple array. I am going to use an array that is 120 long. The reason that it isn't 64 long is to give the board borders that allow you to easily see if a piece
move is valid or not:

```
xxxxxxxxxx
xxxxxxxxxx
xrbnqknbrx
xppppppppx
x........x
x........x
x........x
x........x
xPPPPPPPPx
xRBNQKNBRx
xxxxxxxxxx
xxxxxxxxxx
```

If you can imagine moving a knight looks something like adding or subtracting from it's index, we can see if that actually moves it off the board onto an 'x'.
Without the borders, it would just look like a valid knight move where the night jumps from the a file to the h file for instance!

Now to get started on some actual code, I think we can implement the board representation fairly quick:
```C
/* -=-=-=-=-=-=-=-=-=-=-=-=-= BOARD REPRESENTATION -=-=-=-=-=-=-=-=-=-=-=-=-= */
/* The basis of this board representation is a 10x12 mailbox.                 */
/* I am using uint_fast8_t in an attempt to make this more optimized.         */
/* -=-=-=-=-=-=-=-=-=-=-=-=-=-=-=- PIECE MAP -=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-= */
/* 7 = off the board                                                          */
/* 0 = empty square                                                           */
/*                                                                            */
/* 1  = black pawn                                                            */
/* 3  = black knight                                                          */
/* 4  = black bishop                                                          */
/* 5  = black rook                                                            */
/* 6  = black queen                                                           */
/* 2  = black king                                                            */
/*                                                                            */
/* 9  = white pawn                                                            */
/* 11 = white knight (m'lady)                                                 */
/* 12 = white bishop                                                          */
/* 13 = white rook                                                            */
/* 14 = white queen                                                           */
/* 10 = white king                                                            */
/* -=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-= */
uint_fast8_t board[120] = {
        7,  7,  7,  7,  7,  7,  7,  7,  7,  7,
        7,  7,  7,  7,  7,  7,  7,  7,  7,  7,
        7,  5,  3,  4,  6,  2,  4,  3,  5,  7,
        7,  1,  1,  1,  1,  1,  1,  1,  1,  7,
        7,  0,  0,  0,  0,  0,  0,  0,  0,  7,
        7,  0,  0,  0,  0,  0,  0,  0,  0,  7,
        7,  0,  0,  0,  0,  0,  0,  0,  0,  7,
        7,  0,  0,  0,  0,  0,  0,  0,  0,  7,
        7,  9,  9,  9,  9,  9,  9,  9,  9,  7,
        7, 13, 11, 12, 14, 10, 12, 11, 13,  7,
        7,  7,  7,  7,  7,  7,  7,  7,  7,  7,
        7,  7,  7,  7,  7,  7,  7,  7,  7,  7
};
```
And a quick peak under the hood:
![GDB Board Representation Array](/images/femtochess/gdb_board_array.png "An image of gdb output in the terminal showing the board array in memory")
Our entire board representation done and we're only up to 21 lines of code!
### Move Generation, Evaluation, and Search
Now comes the actual heart of the chess engine. We need to implement minimax with alpha-beta pruning.
## Resources
- [Chess Programming Wiki](https://www.chessprogramming.org/)