---
layout: post
title: "Writing a chess application in Python using python-chess, the Stockfish engine and Prompt toolkit"
author: "Joakim Lönnegren"
categories: blog
tags: [Chess, Python, Stockfish]
image:
  feature: chess-app/feature.png
  teaser: chess-app/teaser.png
---

I've been reading up on some of the early advances in AI and machine learning and stubmle over the [famous chess game](https://en.wikipedia.org/wiki/Deep_Blue_versus_Garry_Kasparov) between Garry Kasparov and Deep Blue. Inspired by this I wanted to get some practice in programming for chess so I started writing a simple chess application where I can play against an engine, in this case Stockfish.

## Setting up
If you don't have stockfish, install it for your distribution. I took the current version from AUR with `yay -S stockfish`.

```fish
mkdir playing-vs-stockfish
cd playing-vs-stockfish
python3 -m venv venv
```

You can check out the current state of this project on [my github](https://github.com/joalon/playing-vs-stockfish).

## Getting started with Stockfish
Apparently Stockfish implements the UCI protocol (Universal Chess Interface). When running `stockfish` in the terminal I got a REPL-like interface which can be verified with the following commands:

```
uci
isready
quit
```

Which prints the UCI commands the engine knows, checks if the engine is ready and finally exits out of the REPL. For the curious, [here's](https://www.shredderchess.com/download.html) a link to the UCI specification download page where the protocol is available as a zipped text file.

Continuing my exploration of UCI, I could also use some other commands to represent and modify the state as well as query for legal moves. I tried the following:

```
position fen r2qk2r/1p1bb1pp/p2pQp2/2p2N2/5P2/1P1PP1P1/PBP4P/R3K3 w Qkq -
go move time 30000
```

The chess stack exchange also has some good info on [working with the UCI protocol](https://chess.stackexchange.com/questions/12580/working-with-uci-protocol-coding)

![Stockfish in terminal](/images/chess-app/first-steps-with-stockfish.jpg)

Now I can get Stockfish to print legal moves and actually play the game. However, I'd like use the [python wrapper](https://github.com/zhelyabuzhsky/stockfish) to control it: `pip3 install stockfish`. Here's a small test I did:

```python
from stockfish import Stockfish

stockfish = Stockfish()
stockfish.set_position(['e2e4', 'e7e6'])

print(stockfish.get_best_move())
print(stockfish.is_move_correct('a2a3'))
```

![Stockfish in python](/images/chess-app/python-stockfish.jpg)

## Implementing parts of the game
To get started playing chess in the terminal I'll start as white, get an input to set as the current position then query the engine for the best move. It will look something like this:

```python
from stockfish import Stockfish

stockfish = Stockfish()
current_position = []

while True:
    while not stockfish.is_move_correct(white_next_move := input("Whites next move: ")):
        pass

    current_position.append(white_next_move)
    stockfish.set_position(current_position)

    black_next_move = stockfish.get_best_move()
    current_position.append(black_next_move)
    stockfish.set_position(current_position)

    print("Black plays: " + black_next_move)
    print("Current position is: " + str(current_position))
```

As you can see, the user interface leaves something to be desired and there's no end condition yet. To make it easier to visualize the board I started looking into FEN strings ([Forsyth-Edwards Notation](https://en.wikipedia.org/wiki/Forsyth–Edwards_Notation)) which is a way to describe the current state of a chess board. For example, the opening position looks like this:

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1
```

The first chunk of text is the encoded positions of the pieces. Lower case letters for black's pieces, an integer for how many empty squares until either another piece or the end of the board and upper case for white's pieces. The ranks are separated by '/'. The other fields are:

```
w             - Whose turn it is 'w' or 'b'
KQkq       - Castling opportunities for both players. 'K' for king side for white, 'q' for queen side for black. '-' if castling is unavailable for both players
-              - En passant square, the square behind a pawn which moved two squares last turn
0              - the number of 
1              -
```

To get this working I had to implement quite a bit of chess in python by myself, which is fiddly and bug prone. For example I have yet to implement castling in the [move function](https://github.com/joalon/playing-vs-stockfish/blob/9beb54d731c225ca84b43536bae7534b698eb52a/play.py#L65):

```python
...
def move(fen_string: str, move: str) -> str:
...
  # Check if the move is castling
  if move == 'O-O':
    pass
  elif move == 'O-O-O':
    pass
...
```

Continuing in the spirit of [Raymond Hettinger:](https://py.checkio.org/blog/5-best-speeches-mr-raymond-hettinger/) "There must be a better way!"

## Replacement using Python-chess
Fortunately, this has already been implemented in python-chess. So, I'll install it `pip3 install python-chess`. Creating a new board and making a couple of moves now looks like:

```python
import chess

board = chess.Board()
print(list(board.legal_moves))

pawn_to_e4 = chess.Move.from_uci("e2e4")
board.push(pawn_to_e4)

print(board)
```

With move validation and the works, success!

## The UI
Now that I have a more or less functional chess I'll get it rendered to the terminal in a nicer way. Let's get started with prompt toolkit `pip3 install prompt-toolkit`. I'll need two window areas: a place to show the board and a place to write the moves. Using a FormattedTextArea to display the board, an HSplit as a separator and an InputBuffer with a callback for making moves, I ended up with something like this:

```python
from prompt_toolkit.application import Application
from prompt_toolkit.document import Document
from prompt_toolkit.key_binding import KeyBindings
from prompt_toolkit.layout.containers import HSplit, Window
from prompt_toolkit.layout.layout import Layout
from prompt_toolkit.styles import Style
from prompt_toolkit.widgets import SearchToolbar, TextArea
from prompt_toolkit.completion import WordCompleter

import chess
import chess.engine

def main():
    search_field = SearchToolbar()

    engine = chess.engine.SimpleEngine.popen_uci('/usr/bin/stockfish')
    board = chess.Board()

    chess_completer = WordCompleter([str(x) for x in board.legal_moves])
    output_field = TextArea(style="class:output-field", text=board.unicode())

    input_field = TextArea(
        height=1,
        prompt=">>> ",
        style="class:input-field",
        multiline=False,
        wrap_lines=False,
        search_field=search_field,
        completer = chess_completer,
        complete_while_typing=True
    )

    container = HSplit(
        [
            output_field,
            Window(height=1, char="-", style="class:line"),
            input_field,
            search_field,
        ]
    )

    def accept(buff):
        new_move = chess.Move.from_uci(input_field.text)
        board.push(new_move)

        result = engine.play(board, chess.engine.Limit(time=0.1))
        board.push(result.move)

        output = board.unicode()
        output_field.buffer.document = Document(
            text = output
        )
        input_field.completer = WordCompleter([str(x) for x in board.legal_moves])

    input_field.accept_handler = accept

    kb = KeyBindings()

    @kb.add("c-c")
    def app_exit(event):
        event.app.exit()
    style = Style(
        [
            ("output-field", "bg:#000044 #ffffff"),
            ("input-field", "bg:#000000 #ffffff"),
            ("line", "#004400"),
        ]
    )

    application = Application(
        layout=Layout(container, focused_element=input_field),
        key_bindings=kb,
        style=style,
        mouse_support=True,
        full_screen=True,
    )
    application.run()

if __name__ == "__main__":
    main()
```

![Prompt UI](/images/chess-app/prompt-ui.jpg)

As you can see I also added a word completer for the input field which grabs the engines legal_moves.

I started implementing a pretty printer which grabs the unicode code points for the different chess pieces, available on [Wikipedia](https://en.wikipedia.org/wiki/Chess_symbols_in_Unicode), however I realized that python-chess already has it covered with `board.unicode()`:

## Summary
I'm pretty proud of how this application turned out and I learned a lot about chess and programming for chess during this exercise. I also think the FEN string concept will be helpful for a reinforcement learning application l'm planning. Until then!

