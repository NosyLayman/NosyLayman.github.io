+++
title = 'Game About Squares Solver'
date = 2022-07-26
[taxonomies]
tags = ["game about squares", "puzzle", "solver", "rust"]

[extra]
katex = true
+++
_A case study of copying the homework badly_

## The game

... you just lost it.

No, the other game. In the title. A [game about squares](http://gameaboutsquares.com/).

I came across this little puzzle on a hellsite, and it immediately rang a bell. In the other day I read a well written blogpost about writing puzzle solvers. It had an example puzzle about some squares that needed to get to their destinations.
<!-- more -->
## The blogpost

So a hysterical digging started; where did I see that post and where did I bookmark it if I ever did that. A good hour later I was reading [the post](https://davidkoloski.me/blog/intelligent-brute-forcing/) again. I can only recommend it, give it a read, well worth the time.

The post introduces a game, and goes through step by step of writing an efficient solver for it. Everytime the puzzle level complexity rises a new trick is revealed from up the sleeve. The code is in Rust, which is convenient for me as I try to learn the language through writing smaller projects.

## The code

First a lot of boilerplate code is needed. The puzzle rules are not mine, so I had to play it a while to get the whole picture. I deemed that 23 levels should be enough experience to be confident no other rules will be introduced later (not that I got stuck on that puzzle, how dare you).

So there is a need for the game state to be represented. It consists of 3 types of game elements (not counting the empty space):

- squares
- goals
- turns

At least this is how they are called in the code, the game narrative is rather reticent about the deep lore.

The first instinct would be using a grid and store all the tiles that are part of the game (as it is in the article), but not only would that be inefficient, but the puzzle is not spatially bound, the squares can freely move and can leave the initial area.

The structures contain only a few data members, and I took the liberty of using 8-bit types knowing that the puzzle space is very limited which actually needs to be used by the solution, and if later a larger input is given I can just upgrade the type then. The thought process was that there will be quite a lot of game states explored while searching the "graph", so better make it as small as it gets.

```rust
#[derive(Copy, Clone, Debug, Eq, Ord, PartialEq, PartialOrd)]
pub struct Pos {
    // using screen coordinates, y increases downwards
    pub x: i8,
    pub y: i8,
}

#[derive(Copy, Clone, Debug, Eq, PartialEq)]
#[repr(i8)]
pub enum Dir {
    Up,
    Down,
    Left,
    Right,
}

#[derive(Debug)]
pub struct Goal {
    pub pos: Pos,
    pub color: i8,
}

#[derive(Debug)]
pub struct Turn {
    pub pos: Pos,
    pub dir: Dir,
}

#[derive(Debug)]
pub struct Square {
    pub pos: Pos,
    pub color: i8,
    pub dir: Dir,
}
```

Okay, now the program can represent the game states, it only needs to get them from somewhere.

## The parser

It was tempting to just hack it together and use some weird and ugly code to read parts of a file and deduce what goes where, but that would have some nightmare syntax on the input and a hell of a lot of complexity that should be avoided. Rust has a great package ecosystem, so I went fishing for a good parser.

What makes a parser good. In my case ease of use was **TOP** priority. Checking about 8 or 9 crates with examples I settled on [Pest](https://pest.rs/). Ok, so I need a grammar and the crate actually generates the parser for me based on that file? How cool is that?!?

The basic idea is to provide every field to every instance of every element in an unambiguous way. I did not want to get verbose, so for the first version it looked like this:

```
A:0,0 v
A:0,1
B:0,2
B:0,3 ^
```

Upper case letter for color, a separator colon (':'), 2 numbers separated by comma, and one of the directions `'^', 'v', '<', '>'` for squares, and without direction for qoals. Turns have no color, so the direction starts the line, then the separator colon and the coordinates. Whitespaces can be added for better readability. `//` style comments are supported.

Some difficulties were encountered how to add the parser crates correctly to enable the code generation first, but it got resolved quickly. Also the grammar initially had a precedence mistake, so whitespaces were swallowed in unfortunate places which also got fixed later.

Now the initial state can be read from file, and the internal representation can be printed
<span style='color:#F43545;'>I</span><span style='color:#FA8901;'>N</span> <span style='color:#FAD717;'>C</span><span style='color:#00BA71;'>O</span><span style='color:#00C2DE;'>L</span><span style='color:#00418D;'>O</span><span style='color:#5F2879;'>R</span>
for debug purposes. Only the solver part is missing ...

## The solver

The game state had all the elements in a vector for each type. When expanding potential states, only the position and direction of the squares change, so all the other elements can belong to a static type, and the state only needs to contain the squares.

```rust
pub struct GameData {
    pub goals: Vec<Goal>,
    pub turns: Vec<Turn>,
}

#[derive(Debug, Default, Clone)]
pub struct State {
    pub squares: Vec<Square>,
}

#[derive(Debug, Default)]
pub struct Game {
    pub data: GameData,
    pub state: State,
}
```

After separating the dynamic and the static data, the solver could examine the possible states reachable from the initial state. Like the good old Breadth-First Search does. The solution is a vector of colors that are the actions needed in order to reach all the goals. If it is achievable.

```rust
pub struct Solver;

impl Solver {
    pub fn solve(initial_state: Game) -> Option<Vec<i8>> {
        None
    }
}
```

Hmm, i said it can examine the potential states...

```rust
pub fn solve(puzzle: Game) -> Option<Vec<i8>> {
    let data = &puzzle.data;
    let initial_state = &puzzle.state;
    let actors_num: i8 = initial_state.squares.len().try_into().unwrap();
    let mut parents = Vec::new();
    let mut queue = VecDeque::new();
    for action in 0..actors_num {
        let next_state = data.action(initial_state, action);
        if data.won(&next_state) {
            return Some(vec![action])
        }
        parents.push((0, action));
        queue.push_back(next_state);
    }

    let mut index = 0;
    while let Some(parent) = queue.pop_front() {
        index += 1;
        for action in 0..actors_num {
            let next_state = data.action(&parent, action);
            if data.won(&next_state) {
                let mut result = vec![action];
                while index != 0 {
                    let (next_index, action) = parents.swap_remove(index - 1);
                    result.push(action);
                    index = next_index;
                }
                result.reverse();
                return Some(result);
            } else {
                parents.push((index, action));
                queue.push_back(next_state);
            }
        }
    }
    None
}
```

If the code shows some resemblance to the code in the blogpost linked is not merely a coincidence. The basic ideas of the solver was almost 1-to-1 transferable to this problem. At least the initial implementation. Only the action is different: not a direction is selected but a color (and I hoped that the color will be a unique identifier throughout the puzzles, and it was). Other difference is that the solver does expand the state to the next possible states it can produce instead of leaving this job to the state (or the game data) producing some results. This was only a personal preference of mine and nothing to do with the problem or the algorithm, I just liked the interface separation better this way.

## The bottleneck

Turns out the intial implementation was enough for a few levels, but soon hit a scalability problem. Compared to the other game, the choices scale with the number of squares. The other had 4 directions, so the states explode with this factor $\mathcal{O}(4^n)$ where this scales with $\mathcal{O}(k^n)$ with k=1 being trivial, k>4 meaning extremely bad news.

So cheating the next step off of the A+ student, the explored states need some serious reductions. This is achieved by adding a hashset, so we know what was already explored, and don't bother expanding or evaluating again. This means every element present in the game state needs to be hashable.

```rust
#[derive(/*...*/, Eq, PartialEq, Hash)]
```

All the way down.

The only difference in the solver is checking in a hashset whether all the expensive logic is required, or the state can be skipped.

```rust
let mut states = HashSet::new();
// same old code
while let Some(parent) = queue.pop_front() {
    index += 1;
    if !states.contains(&parent) {
        for action in 0..actors_num {
	    // same old code
        }
        states.insert(parent);
    }
}
```

This had enough steam to go all the way to puzzle #17. This was the point my patience ran out with manually translating the inputs to the stupid format I came up with, and manually translating back the output to the actual puzzle.

## The format change

So using consecutive letters from 'A' to describe square colors are arbitrary and hard to follow. The initial puzzle state can be translated pretty easily algorithmically, just name the first color you encounter 'A'. Or the first square's color. While playing back the solution acquired in a form of an alphabet soup and the game not even looking like how it started it is hard to maintain the mapping in mind.

New parser rules were added: color is any word that starts with a capital letter. That's it. Puzzle #1 now looks like this.

```
Blue:0,0 v
Blue:0,1
Red:0,2
Red:0,3 ^
```

This meant some bookkeeping of the colors is needed to still be able to represent them as small integers.

```rust
pub struct GameData {
    pub goals: Vec<Goal>,
    pub turns: Vec<Turn>,
    pub color_map: Vec<String>,
}
```

And the mapping was updated as encountering the colors in order.

```rust
fn to_color(color : &str, vec : & mut Vec<String>) -> i8 {
    if let Some(pos) = vec.iter().position(|e| e == color) {
        return pos as i8;
    }
    vec.push(color.to_string());
    (vec.len() - 1) as i8
}
```
And the signature of the solver is also adapted.

```rust
pub fn solve(puzzle: Game) -> Option<Vec<String>>
```

With ease the puzzles were encoded, solved and replayed in the browser.

Until level #31. Which took an eternity to compute.

## The constraint

In the article which was blatantly copied, there is a step sorting the game state to make a canonical representation of every possible state that means the same in puzzle state. The colored squares are interchangeable there, which problem does not arise here. The action that takes an initial square that is acted upon does not reorder the vector, just updates the positions and directions where necessary. This only would've been a problem for the hashset by allowing the same state represented in different permutations to get processed again.

In the spirit of reducing the unnecessary exploration of states one trick I came up with was just a simple limitation. In each puzzle the squares can leave the initial bounding area, and this move is often required by the solution (pushing a square out which is directed backwards to later push back another square). Let's observe that  it is redundant for any of the squares to leave the initial area farther than the number of squares in the puzzle. This may be a valid move, but cannot contribute to a shortest solution.<sup>\*</sup> This can reduce the explored states where squares just go rogue and escape early potentially without means to go back.

<sub>_<sup>\*</sup> Proving it is left as an excercise to the reader..._</sub>

```rust
pub struct Area {
    tl: Pos,
    br: Pos,
}

impl Area {
    pub fn new(tl: &Pos, br: &Pos) -> Area {
        assert!(tl.x <= br.x);
        assert!(tl.y <= br.y);
        Area {tl: *tl, br: *br}
    }
    pub fn inside(&self, pos: &Pos) -> bool {
        self.tl.x <= pos.x && pos.x <= self.br.y && self.tl.y <= pos.y && pos.y <= self.br.y
    }
    pub fn tl(&self) -> Pos {
        self.tl
    }
    pub fn br(&self) -> Pos {
        self.br
    }
}
```

This represents the original area of interest outside of which is not a valid move for the sake of reducing complexity.
Area computing code was already present for the debug drawing, now properly encapsulated.

```rust
pub fn get_area(state: &State, data: &GameData) -> Area {
    assert!(!state.squares.is_empty());

    let mut tl = state.squares[0].pos;
    let mut br = tl;
    for e in &state.squares {
        tl.x = min(tl.x, e.pos.x);
        tl.y = min(tl.y, e.pos.y);
        br.x = max(br.x, e.pos.x);
        br.y = max(br.y, e.pos.y);
    }
    for e in &data.goals {
        tl.x = min(tl.x, e.pos.x);
        tl.y = min(tl.y, e.pos.y);
        br.x = max(br.x, e.pos.x);
        br.y = max(br.y, e.pos.y);
    }
    for e in &data.turns {
        tl.x = min(tl.x, e.pos.x);
        tl.y = min(tl.y, e.pos.y);
        br.x = max(br.x, e.pos.x);
        br.y = max(br.y, e.pos.y);
    }

    Area{tl, br}
}
```

In the solver this needs to be incorporated to filter out the runaway squares. Be there AND be square... 

```rust
pub fn solve(puzzle: Game) -> Option<Vec<String>> {
    let data = &puzzle.data;
    let initial_state = &puzzle.state;
    let area = get_area(initial_state, data);
    let mut tl = area.tl();
    let mut br = area.br();
    let num_sq = initial_state.squares.len() - 1;
    tl.x-=num_sq as i8;
    tl.y-=num_sq as i8;
    br.x+=num_sq as i8;
    br.y+=num_sq as i8;
    let area = Area::new(&tl, &br);
```

The area is expanded by the number of squares. After some thinking it got reduced by one. The reason: no meaningful change can come when all the squares are outside (pushing eachother out then back) because what if there were any turns or goals there? Then the initial area would be bigger. The last meaningful state change (required for a solution) happens inside the boundaries of the initial area, that means only N-1 squares can be outside for that move to make it make sense.

The filtering happens before adding to the next explorable states. So every state that has a bad move is computed, checked, and discarded, which is better than pursuing it further.

```rust
if data.won(&next_state) {
    // same old code
} else {
    if next_state.squares.iter().any(|sq| !area.inside(&sq.pos)) {
        continue;
    }
    parents.push((index, action));
    queue.push_back(next_state);
}
```

This made the runaway squares problem disappear, and fast enough for the remainder of the puzzles to be completed.

## The conclusion

There are many other areas that could be optimized or algorithmically improved. Actually, there is a LOT of room for improvement, but the problem at hand is solved isn't it?

I encourage everyone to read the original (and much more quality) article to the end, and learn from it.

The code is up on [github](https://github.com/NosyLayman/gameaboutsquares_solver), if You have any idea to improve it, or just want to play with it.
