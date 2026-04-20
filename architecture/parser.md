# OS128 Minishell Parser Detailed Design

This document describes the parser implementation in the OS128 minishell, which uses `lib/tokenize.asm` for tokenization and `lib/matchword.asm` for command matching.

## 1. Parser architecture

The minishell parser is implemented across three main components:

- `kernel/minishell.asm` — main shell loop and command dispatch
- `lib/tokenize.asm` — input tokenization and parsing utilities
- `lib/matchword.asm` — keyword table matching for command recognition

The parser processes user input from the readline buffer, tokenizes it into words, matches the first token against a command table, and dispatches to the appropriate command handler.

## 2. Input tokenization (`lib/tokenize.asm`)

### 2.1 Tokenization process

The tokenizer breaks the input line into tokens separated by whitespace (space character).

Key functions:

- `tokenize` — main tokenization entry point
- `next_token` — extract next token from input
- `tokenizer_reset` — reset tokenizer state

### 2.2 Tokenization algorithm

1. `tokenize` validates the readline buffer exists
2. Sets separator to space (`' '`)
3. Calls `next_token` to extract tokens

`next_token`:

- Starts from `tok_next` position in readline buffer
- Scans forward until separator or end of buffer
- Calculates token length
- Returns token start position (Y) and length (X)
- Updates `tok_next` for next call

### 2.3 Tokenizer state

- `tok_pos` — current position within current token
- `tok_next` — position to start next token search
- `separator` — character used to separate tokens (default space)

### 2.4 Parsing utilities

Additional parsing functions:

- `get_char` — get next character from current token
- `get_hex_byte` — parse hex byte from token
- `get_num_byte` — parse decimal byte from token
- `get_byte` — parse byte value from token

These use `next_token` to advance through tokens and parse specific data types.

## 3. Command matching (`lib/matchword.asm`)

### 3.1 Matching process

`match_word` searches a keyword table for commands matching the input token.

Key function:

- `match_command` — main matching entry point

### 3.2 Keyword table format

The keyword table is a structured array:

```asm
shell_commands:
    !word .command_name
    !word command_handler_address
    !byte 0 ; end marker
```

Each entry consists of:

- Null-terminated command string
- 16-bit handler address
- Zero byte to terminate the table

### 3.3 Matching algorithm

1. `match_command` takes token length (X) and start position (Y)
2. Iterates through keyword table entries
3. Compares input token against each command string
4. Returns command index and handler address if match found

### 3.4 Matching logic

- Uses `search_status` bit to track partial matches
- Compares characters one-by-one
- Handles partial matches by advancing through input
- Requires exact match up to command length
- Ensures command ends at word boundary (null terminator)

### 3.5 State variables

- `keyword_tab_ptr` — pointer to keyword table
- `keyword_tab_pos` — current position in table
- `keyword_idx` — current command index
- `search_pos` — current position in input token
- `found_length` — length of partial match

## 4. Minishell command dispatch (`kernel/minishell.asm`)

### 4.1 Shell loop

The main shell loop in `minishell_create`:

1. Sets up signal handlers
2. Points to command table (`shell_commands`)
3. Reads input line via readline
4. Calls parser to process input

### 4.2 Parsing sequence

1. `tokenizer_reset` — reset tokenizer state
2. `tokenize` — get first token (command)
3. `match_word` — match against command table
4. If match found, dispatch to handler
5. Handler may parse additional tokens using tokenizer functions

### 4.3 Command execution

For matched commands:

- `call_command` creates a new thread for the command
- Replicates tokenizer state to new thread
- Passes device number and input stream
- Command thread runs independently

### 4.4 Error handling

- Invalid commands return to shell loop
- Thread creation failures retry with backoff
- Signal handling for interrupts and termination

## 5. Data structures

### 5.1 Readline buffer

- `readline_buffer` — input line buffer
- `readline_pos` — current position in buffer
- `readline_ptr` — pointer to buffer

### 5.2 Tokenizer state

- `tok_pos` — position in current token
- `tok_next` — start of next token
- `separator` — token separator character

### 5.3 Command table

- Structured array of command entries
- Each entry: string + handler address
- Terminated by zero byte

## 6. Example parsing flow

1. User enters: `ls -l /dev`

2. `readline` stores input in buffer

3. `tokenizer_reset` initializes state

4. `tokenize` extracts "ls" (length 2, position 0)

5. `match_word` searches command table for "ls"

6. Match found, returns handler address

7. Command handler runs in new thread

8. Handler calls `next_token` for "-l"

9. Handler calls `next_token` for "/dev"

10. Handler processes arguments and executes

## 7. Key code references

- `kernel/minishell.asm` — `.shell_loop`, `.execute_command`, `.call_command`
- `lib/tokenize.asm` — `tokenize`, `next_token`, `get_hex_byte`, `get_num_byte`
- `lib/matchword.asm` — `match_command`, keyword table scanning

## 8. Summary

The OS128 minishell parser uses a two-stage approach: tokenization breaks input into words, and command matching identifies valid commands against a keyword table. This design allows extensible command sets with efficient parsing of arguments and parameters.
