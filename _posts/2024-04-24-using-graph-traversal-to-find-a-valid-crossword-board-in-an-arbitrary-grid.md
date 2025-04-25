---
layout: post
title: "Using Graph Traversal To Find a Valid Crossword Board In an Arbitrary Grid"
date: 2024-04-24
categories: [algorithms, graph theory, puzzles]
author: Your Name
---

## What Is xworder.io?

[Xworder](http://xworder.io) is a system I built that procedurally generates crossword puzzles without any human intervention. A new crossword is displayed on xworder.io every day. Currently there are three stages to generating a crossword:

1. Building an empty board, set with closed and open (black and white) cells,
2. Finding a set of words or phrases that will fit the board,
3. Generating clues for these words so that the puzzle is solvable.

This article is about step 1, which I implemented after already having developed steps 2 and 3.

## Why Generate A Board

In the course of building Xworder I improved the algorithm for solving an empty board (more in this in a later post) such that it could, in a reasonable amount of time, fill boards larger than the size of a mini, usually 5x5. Prior to this I had been manually writing the boards myself, e.g.:

[Image of manual board]

but as the boards the algorithm was filling increased in size, this manual approach became tedious, and worse, unscalable. In order to solve this problem I therefore set out to write a script that could generate a random, empty board made up of open and closed cells within the bounds of a set of constraints.

## Exploring the Problem

Stated more formally, the problem I set out to solve was: Create an MxN array of arrays where M is the number of rows, each an array, and N is the number of columns, each a string that is either open (' ') or closed ('#').

As I dug into an implementation, however, I realized this conception of the problem was insufficient, since it could produce something that looked like this:

[Image of invalid board]

This empty board is obviously unsuitable, but why? Thinking things through, I realized that:

1. the board needed to be rotationally symmetrical to match the convention for these puzzles,
2. the board needed to specify a ratio of closed and open cells and,
3. critically, the board needed to define a minimum size for any wordspace (a contiguous row or column of open cells).

In other words, the board needs to look the same when you spin it 180 degrees, it needs to have enough open cells to be interesting to solve, and any contiguous, vertical or horizontal set of blank cells (where we would put a word) needs to be some minimum length.

Once these extra constraints are taken into account, we have a sufficiently strict definition of the problem to proceed about implementing a solution that will give us boards that anyone would enjoy solving.

## Implementing the Algorithm

The first step in generating a board per the specifications is to instantiate it and figure out how many of the cells will be closed. The floor of M × N × ratio_closed will give us the number of cells (for example, a 13 by 13 board where 30% of the cells should be closed produces floor(13 × 13 × .3) = floor(50.7) = 50.

Once we've built out an MxN array of arrays of empty strings, we could simply select cells at random and flip them until we've reached the closed count. If we do this, however, we have no guarantee that the word spaces will be an appropriate length or that the board will be symmetrical. How do we satisfy these guarantees?

### Symmetry

The solution to this problem is relatively straightforward. Whenever we decide upon a cell to close, we can also close the cell at the "inverted" coordinates: M - x - 1 and N - y - 1. For example, if we are inserting at coordinates 3, 5 on a 15x15 board, the inverted coordinates are 15 - 3 - 1 and 15 - 5 - 1, (11, 9).

[Image of symmetrical board]

If the center cell in a board has been selected, then the inverse is also the center. Although only a single cell is closed in this case, this is acceptable because rotational symmetry has been preserved.

### Wordspace Size

The question of how to keep the word spaces above a certain length is harder to solve than the question of symmetry. Scanning from side to side and up and down in order to figure out the size of existing wordspaces would tell us which ones are too small, but wouldn't tell us which cells are viable for closing. Furthermore, since two wordspaces with different lengths can cross in a single open cell (one in a row, the other in a column), should the vertical or horizontal wordspace length be reflected in the cell where they intersect? This section is displayed in the section of the crossword below: the orange is a wordspace of length 5, while the blue is a wordspace of 3 - which value should we use?

[Image of intersecting wordspaces]

The key insight here is that we aren't really concerned about the size of a given wordspace, but rather, about an open cell's distance from the closest closed cell, whether row- or column-wise. If we can figure out how far each open cell is from a closed cell, then we can simply select any cell where the distance is greater than the minimum length we've specified as a constraint.

How do we get this distance count for all open squares in a grid? The answer lies in realizing that the grid is in fact a graph, and that if we perform a breadth-first traversal starting from all of the closed cells, each "level" of the traversal is an extra unit of distance. By completing a traversal and recording the level in each cell as it is traversed, we can build a heat map of the board and determine the set of cells that are candidates for closing!

In pseudocode, then, we can use the following algorithm:

```python
SET grid = [ … ] # an array of array of strings representing closed and open cells
SET get_closed_cells = FUNCTION () { … } # this method loops over all of the cells and collects the coordinates of closed cells

# Initialize all non-closed cells to 'open'
FOR EACH row in rows:
    FOR EACH cell in row:
        IF cell is not closed:
            SET cell to open

# Initialize BFS variables
SET distance = 0
SET queue = get_closed_cells(rows) # starts the traversal from the closed cells
SET seen = empty set
SET grid_size = min(length(rows), length(rows[0]))

# Handle case where no closed cells exist
IF queue is empty:
    FOR EACH cell in grid:
        SET cell value to grid_size

SET next_queue = empty list
SET seen_neighbors = empty set

# Main BFS loop
WHILE queue is not empty:
    # Process current cell
    (row, col) = dequeue from queue
    ADD (row,col) to seen set
    
    IF cell at (row,col) is open:
        SET cell value to current distance
    
    # Process neighbors
    FOR EACH neighbor of (row,col):
        neighbor_signature = string representation of neighbor coordinates
        
        IF neighbor_signature is in seen_neighbors:
            CONTINUE
        
        IF neighbor cell is open:
            ADD neighbor_signature to seen_neighbors
            ADD neighbor to next_queue
    
    # Level completion check
    IF queue is empty:
        SET queue = next_queue
        SET next_queue = empty list
        INCREMENT distance
```

Running this algorithm for a 17 x 17 size board with a minimum wordspace length of 3 we could produce the following:

[Image of board with algorithm]

Looks good… or does it?

## The Final Piece

Although the board has rotational symmetry the wordspace length is still wrong at the edges - looking closely we can see that many of the wordspaces near the border are two or even one space rather than the required three.

After reflecting I understood the problem: the algorithm isn't treating the edges of the board as bounds of the wordspace. As a result, any wordspace that crosses the edge – from the algorithm's perspective – has infinite cells added to it. In order to solve this problem, a simple solution presented itself: close all of the cells on the outer edge of the board prior to running the algorithm.

Starting with a blank board with the top and bottom rows and left- and rightmost columns closed, the algorithm runs as it should:

[Image of final board]

Success!

## Conclusion

Solving this problem was satisfying because it unlocked the ability to generate larger crosswords at scale and automated a pretty painful manual process. Beyond this though it was cool to apply a concept that had been largely academic – breadth-first-traversal – to a real-world problem and see how conceptualizing something as a graph really did produce real solutions, and solutions that are often visually striking and pretty.


