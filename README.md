# Brainfuck In Hexagony

A brainfuck interpreter written in [Martin Ender's Hexagony](https://github.com/m-ender/hexagony).

# Specs

Takes brainfuck source code as a command line argument, and input to the brainfuck program as an optional second argument.

Memory is unlimited in both directions from the origin.

Cells are 8-bit wrapping.

Reading a byte after input is exhausted reads a zero byte.

# Explanation


![Execution map of Hexagony code](https://github.com/MeWhenI/Brainfuck-In-Hexagony/blob/main/images/Code_Map.png "Code map")

Images made using [Timwi's Hexagony Colorer](https://github.com/Timwi/HexagonyColorer). 

The code makes use of Hexagony's multiple IP's to tackle different parts of the problem, below is a breakdown of what each IP does.

### IP 1 - Equality subroutine
![IP 1 of Hexagony code](https://github.com/MeWhenI/Brainfuck-In-Hexagony/blob/main/images/IP_1.png "IP 1")

Starts in NE corner moving SE.

Hexagony doesn't have a builtin to test equality, so this IP serves as a callable subroutine which tests whether 2 numbers are equal, returning 0 if they are and 1 otherwise. Specifically, it computes `(x - y) * (x - y) == 0 ? 0 : 1`. When it finishes, it returns to IP 0.

### IP 3 - Cell wrapping subroutine
![IP 3 of Hexagony code](https://github.com/MeWhenI/Brainfuck-In-Hexagony/blob/main/images/IP_3.png "IP 3")

Starts in SE corner moving W.

A helper subroutine that contains some code which would otherwise appear in multiple places. Modulos a memory cell by 256 and then calls IP 1 to check whether the cell is equal to 0.

### IP 0 - Create instruction stack
![IP 0 of Hexagony code](https://github.com/MeWhenI/Brainfuck-In-Hexagony/blob/main/images/IP_0.png "IP 0")

Starts in NW corner moving E.

Building data structures across multiple cells in Hexagony is hard, but since it has arbitrary precision integers, this can be circumvented by storing an entire array in a single number. For example, consider the int array `[v, w, x, y]` with cell size `cs`. This can be stored in an integer as `arr = v + cs * (w + cs * (x + cs * (y) )`. Then, accessing the value at index `i` in `arr` can be done by calculating `arr / pow(cs, i) % cs`. For the instruction array I use cell size 10 because its enough space to store 8 instruction types plus EOF, and because the `0` command makes powers of 10 easy in Hexagony. A nice result of using a cell-size 10 array stored in an integer is that each digit holds 1 value. So for example, `[1, 2, 3, 4]` becomes `4321` (it looks reversed because the smallest digit holds the first value).
 With those ideas in mind, IP 0 does the following:
 
 Starting from the black path, enters a loop that reads each byte from input until it reaches EOF (a null byte). For each byte:
 
 The green path checks whether the byte is a valid brainfuck instruction, any of `+,-.<>[]`. It does this by checking `eq((b+1)/4,11)+eq(b,60)+eq(b,62)+eq(b,91)+eq(b,93)`, where eq is a subroutine implemented in IP 1 that returns 0 if its operands are equal, otherwise 1. 
 
 If b is not a valid instruction, it moves to the yellow path, which skips to the next byte. Otherwise, it moves to the blue path.
 
 The blue path takes the instruction's ASCII value `b` and calculates `b % 30 % 9 + 1`. This hashes each instruction type to a unique one digit value:
 
 | Operator | ASCII | Hash |
 | -------- | ----- | ---- |
 |  +       | 43    | 5    |
 |  ,       | 44    | 6    |
 |  -       | 45    | 7    |
 |  .       | 46    | 8    |
 |  <       | 60    | 1    |
 |  >       | 62    | 3    |
 | \[       | 91    | 2    |
 | \]       | 93    | 4    |

It then appends this hashed value to our array. The array building works like this:
 
`arr` starts at zero, and a value `base` starts at 1. To append a value `x` to `arr`, we add `x * base` to `arr`, and then multiply `base` by 10.
 
When IP 0 finishes, it has created an instruction stack out of the inputted brainfuck code. For example, an input of `+++.>,.` produces `8638555` (see that this looks like the input reversed and then each byte mapped onto the hash table).

It then jumps to IP 5, and actually enters into an infinite loop so that anytime IP 0 is called it will automatically jump to IP 5. The reason for this is that IP 1, the equality testing subroutine, is set to always return to IP 0. Since I want to reuse IP 1 later and have it return to IP 5, and am finished with IP 0, the easiest solution is to hardwire IP 0 to always redirect to IP 5.

### IP 5 - Main loop
![IP 5 of Hexagony code](https://github.com/MeWhenI/Brainfuck-In-Hexagony/blob/main/images/IP_5.png "IP 5")

Starts in W corner moving NE.

Starts at the black path, initializing a pointer for the brainfuck source, `code_ptr` to 1. Note that this points to index 0, since we're indexing our array by powers of 10 and 10 ^ 1 = 0.

At this point, we have the following structure in our Hexagony memory:

![Memory Visualization](https://github.com/MeWhenI/Brainfuck-In-Hexagony/blob/main/images/Mem_map.jpg "Hexagony Memory")

The code stack and pointer are stored right beside eachother, and 2 rows below them is the brainfuck memory tape. Because the only other data that needs to be stored between instructions is the code stack and pointer, we can simply carry them with us as we move left or right across memory. So to execute the `<` brainfuck instruction we copy them into the pink section, and to execute the `>` instruction we copy them into the orange section.

The light blue is the main loop. It grabs the value in `code_stack` indexed by `code_ptr` and moves to the corresponding red path. Afterwards, it increments `code_ptr` (multiplies it by 10). Once the end of `code_stack` is reached, ie. `code_stack / code_ptr == 0`, it takes the termination path and the program exits.

The "increment", "decrement", "read byte", and "write byte" paths perform the corresponding action on the memory cell under the code stack. None of them cause the cell to wrap because wrapping is only important in brainfuck when checking a while condition or printing, and Hexagony already prints bytes modulo 256, so we only need to wrap at while loops. Also, for read byte: Hexagony will in some instances read a 0 the first time it reads a byte past EOF and -1 all subsequent times. To standardize this behavior, the read byte path replaces any -1 it reads with 0.

The "Move left" and "Move right" paths move through Hexagony's memory, carrying `code_stack` and `code_ptr` with them.

"Start While" and "End While": Each of these paths starts by calling IP 3, a helper subroutine that runs some code that would otherwise be redundant between both paths. It modulos the current memory cell by 256 and then calls the IP 1 equality subroutine to test whether the cell is now equal to 0. Then the "End While" path inverts the result and adds 5, or the "Start While" path adds 4. This results in either a 4 or a 5 based on whether the memory cell is zeroed and whether we're on the "Start While" or "End While" path:
 |          | Start `[` | End `]` |
 | :------: | --------- | ------- |
 | mem == 0 | 5         | 4       |
 | mem != 0 | 4         | 5       |

 Then the "Go to IP" Hexagony instruction is run, which acts as a no-op if the value is 5, otherwise takes us to IP 4, the "Jump to the corresponding While instruction" subroutine.
 
### IP 4 - Jump to the other side of a While loop
![IP 4 of Hexagony code](https://github.com/MeWhenI/Brainfuck-In-Hexagony/blob/main/images/IP_4.png "IP 4")

Starts in SW corner moving NW.

Whenever a square bracket is reached in brainfuck, we may need to jump to its corresponding paired square bracket. This IP handles that.

It starts at the leftmost part of the yellow path, where it makes a copy of the current instruction's hash value (2 for `[`, 4 for `]`, see the hash table above) and subtracts 3. This gives us -1 for `[` and 1 for `]`. Call this value `bracket_offset`. A loop is entered that checks at each iteration whether `brack_offset` has reached 0. If it has, the orange path is taken and the subroutine ends. Otherwise it starts down the purple path. The purple path either increments or decrements `code_ptr` (actually multiplies or divides by 10) depending on which direction we need to be moving to find the corresponding While instruction, then checks whether the new current instruction is a While instruction. If it is, then it adds -1 to `brack_offset` for `[`, or 1 for `]`. The result is that `brack_offset` reaches zero once the number of brackets we have reached is balanced; in other words, once the corresponding While instruction has been reached.

# Golfed version

Coming soon (?)
