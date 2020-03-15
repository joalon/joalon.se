---
layout: post
title: "Playing Tic tac toe versus a Reinforcement learning agent"
author: "Joakim Lönnegren"
categories: blog
tags: [Reinforcement learning, Python, Tic tac toe]
image:
  feature: rl-tictactoe-agent/feature.png
  teaser: rl-tictactoe-agent/teaser.png
---

I've been reading the first few chapters of the book [Reinforcement Learning](https://www.amazon.com/Reinforcement-Learning-Introduction-Adaptive-Computation-ebook/dp/B008H5Q8VA) and thought I'd try my hand at implementing the algorithm for training a simple agent which the authors describe in the first few chapters.

You can check out the code on [my Github](https://github.com/joalon/rl-tictactoe-agent).

## Implementing the game
First I needed to actually write the game the agent was going to play. I chose tic-tac-toe since it's an easy game which can be extended to later play on a 5-by-5 board which offers more tactical possibilities and doesn't draw as easily as the 3-by-3.

Enter the class TicTacToe which holds the board state and has methods to manipulate it. The board is basically just a list of lists: `board = [[".", ".", "."], [".", ".", "."], [".", ".", "."]]`. I chose to represent state as something like FEN-position notation in chess, the first state is `3/3/3` and can look like this after a few moves: `1o1/xox/1x1`, and I 

![TicTacToe state example](/images/rl-tictactoe-agent/state-example.jpg)

This is encapsulated in the `TicTacToe` class, which has a move function as well as `game_ended` function which is called when a move ends in a winning/losing state. I put this class into a tictactoe.py module.

To play it I added a HumanAgent class (inherits from TicTacToeAgent) which asks the player for input. These live in their own module, agents.py.

```python
class TicTacToeAgent:

    def __init__(self, board):
        pass

    def emit_move(self):
        pass

    def notify_game_ended(self, won: bool):
        pass

class HumanTicTacToeAgent(TicTacToeAgent):
    """
    A tictactoe agent that asks for input
    """

    def __init__(self, board):
        self.playing_board = board

    def emit_move(self):
        move = input("Make a move: ")
        move_x, move_y = move.split(" ")
        return (int(move_x), int(move_y))

    def notify_game_ended(self, won: bool):
        pass
```

With two of these I was able to play against myself:

```python
from tictactoe import TicTacToe
from agents import HumanTicTacToeAgent

tictactoe = TicTacToe()
Player1 = HumanTicTacToeAgent(tictactoe)
Player2 = HumanTicTacToeAgent(tictactoe)

print(tictactoe)
while not tictactoe.game_has_ended()[0]:
    next_move1 = Player1.emit_move()
    tictactoe.move(next_move1)
    print(tictactoe)
 
    if not tictactoe.game_has_ended()[0]:
        next_move2 = Player2.emit_move()
        tictactoe.move(next_move2)
        print(tictactoe)
```

![Gif of playing vs myself](/images/rl-tictactoe/playing-vs-myself.gif)

## Add reinforcement learning
Now that I had a functioning game I needed an agent to play against. The first one I wrote was a random player:

```python
from random import choice

def RandomTicTacToeAgent(TicTacToeAgent):
     def __init__(self, board):
         self.board  = board

     def emit_move(self):
         valid_moves = self.board.get_valid_moves()
         return choice(valid_moves)

     def notify_game_ended(self):
         pass
```

At last an opponent! But not a very good one, next up was implementing reinforcement learning. The basic algorithm is this:

1. Assume every intermediate game state is initialized with a value of 0.5 and any winning state to 1 and losing state to 0
2. Play a game and pick the actions leading to the highest value state. Store the intermediate states
3. Increase each intermediate states value if the agent won, otherwise decrease it
4. repeat from nr 2

There are lots of customizations available but I started implementing the basic one.

```python
class RLTicTacToeAgent(TicTacToeAgent):
    """
    A tictactoe agent that makes moves according to a reinforcement learning algorithm
    """

    def __init__(self, board, moves_dict):
        self.playing_board = board
        self.moves_dict = moves_dict
        self.exploration_rate = 0.1
        self.visited_states = []
        self.reward = 1.0

 def emit_move(self):

        valid_moves = self.playing_board.get_valid_moves()

        if uniform(0, 1) <= self.exploration_rate:
            chosen_move = choice(valid_moves)

            visited = TicTacToe(self.playing_board.get_state())
            visited.move(chosen_move)
            next_state = visited.get_state()
            self.visited_states.append(next_state)

            if next_state not in self.moves_dict.keys():
                self.moves_dict.update({next_state: 0.5})

            return chosen_move

        else:
            considering_moves = []

            for move in valid_moves:
                next_board = TicTacToe(self.playing_board.get_state())
                next_board.move(move)
                next_state = next_board.get_state()

                if next_state in self.moves_dict.keys():
                    considering_moves.append({move: self.moves_dict[next_state]})
                else:
                    self.moves_dict.update({next_state: 0.5})
                    considering_moves.append({move: 0.5})

            sorted_list = list(
                reversed(
                    sorted(considering_moves, key=lambda dict: list(dict.values())[0])
                )
            )

            chosen_move = list(sorted_list[0].keys())[0]
            next_board = TicTacToe(self.playing_board.get_state())
            next_board.move(chosen_move)
            next_state = next_board.get_state()
            self.visited_states.append(next_state)

            return chosen_move

    def notify_game_ended(self, won: bool):
        reward = self.reward / len(self.visited_states) if won else -(self.reward /len(self.visited_states))
        for v in self.visited_states:
            self.moves_dict[v] += reward
```

The first game this agent plays will essentially be random since it only picks actions which leads to states with a value of 0.5. When it has played some games and has updated the values it will start to adapt to the game, though.

To sum up I have the following directory:

```
rl-tictactoe-agent/
   agents.py
   tictactoe.py
   play.py
   data.p
```

## Training
With the algorithm done I started training it. I added 'training.py' which takes a[^1]esult during training, if enabled. Since the program holds the state in a pickled file (data.p) I ran the training program `time python3 training.py --file=data.p --verbose -n=100000`

![Executing training program](/images/rl-tictactoe-agent/executing-training.jpg)

And after these 100 000 rounds (and ~15 minutes) it seems to have learned a thing or two. When playing against itself it draws all the time, meaning it either plays very bad or it has [solved the game](https://en.wikipedia.org/wiki/Tic-tac-toe#Strategy).

![List of draws](/images/rl-tictactoe/list-of-draws.jpg)

To take a look at the state dict I ran `python3 -i training.py --file=data.p`, which put me in an interactive python shell after the training run, and then print it with `state_dict`.

## Summary
I'd like to refactor this in the future by moving the game logic into static methods, but that is for another weekend project. Most of all I'm glad the agent worked and finally beat me in a game of tic-tac-toe!
