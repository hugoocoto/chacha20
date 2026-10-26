# ChaCha20 in hardware

A ChaCha20 stream cipher ([RFC 8439](https://www.rfc-editor.org/info/rfc8439/))
built in [Digital](https://github.com/hneemann/Digital). It encrypts one 32-bit
word per clock. Decryption is the same operation.

## How it works

ChaCha20 turns the key, a nonce and a block counter into 64 bytes of
keystream, and XORs them with the data. The circuit `core.dig` does this in
three parts:

- **Block function** (`block_function.dig`): computes a 64-byte keystream
  block. It has one double round in hardware and runs it 10 times, one per
  clock, so a block takes 10 clocks.
- **Keystream register**: holds the 16 words of the current block. A word
  counter picks one word per clock, and it's XORed with the input.
- **Control** (`ctrl.dig`): a state machine with `IDLE`, `GEN` (waiting for
  the first block) and `XOR` (encrypting).

While the 16 words of a block are used, the block function already computes
the next one. That one is ready after 10 clocks, before the 16 words run
out, so the data never has to wait, except at the start of a message.

## Inputs and outputs

Data words move with a valid/ready handshake: a word is transferred on a
rising clock edge when both `valid` and `ready` are 1.

| Input | Bits | |
| --- | --- | --- |
| `CLK` | 1 | Clock |
| `rst` | 1 | Reset (back to `IDLE`) |
| `start` | 1 | Start a message |
| `k0`..`k7` | 32 | Key (256 bits) |
| `n0`..`n2` | 32 | Nonce (96 bits) |
| `counter0` | 32 | First block counter (usually 1) |
| `in_data` | 32 | Plaintext word |
| `in_valid` | 1 | `in_data` holds a word |
| `in_last` | 1 | Last word of the message |
| `in_keep` | 4 | Valid bytes of the word (`1111` = all four) |
| `out_ready` | 1 | The receiver accepts `out_data` |

| Output | Bits | |
| --- | --- | --- |
| `in_ready` | 1 | The circuit accepts `in_data` |
| `out_data` | 32 | Ciphertext word |
| `out_valid` | 1 | `out_data` holds a word |
| `out_last` | 1 | Copy of `in_last` |
| `out_keep` | 4 | Copy of `in_keep` |

All words are **little-endian**: `k0` holds key bytes 0..3, with byte 0 in
bits 7..0. The same goes for the nonce and the data.

## How to use it

1. Set the key, nonce and `counter0`, and keep the key and nonce fixed until
   the message ends.
2. Hold `start` for one clock.
3. Send the message words through `in_data` / `in_valid`, and set `in_last`
   on the last one. If the last word isn't full, `in_keep` says which bytes
   are valid. The ciphertext comes out on the same clock as each input word.
4. After the last word the circuit goes back to `IDLE`, ready for another
   `start`.

`in_ready` stays at 0 while the first block is computed. After that it
follows `out_ready`. With no pauses, a message of `W` words takes
**10 + W clocks** from `start`.

Example from RFC 8439 §2.4.2: key `00 01 02 … 1f`, nonce
`00 00 00 00 00 00 00 4a 00 00 00 00`, `counter0 = 1`, and the first word of
the text "Ladies and…" is `0x6964614c`. The first word that comes out is
`0x9a352e6e`. This is the test in `core.dig`.

## Tests

Each `.dig` file has a test. Run them with:

```sh
for f in *.dig; do java -cp Digital.jar CLI test "$f"; done
```

## References

- https://www.rfc-editor.org/info/rfc8439/
- https://en.wikipedia.org/wiki/ChaCha20-Poly1305
- https://lnkd.in/p/emWu7Mmf
- https://github.com/hneemann/Digital
- https://github.com/hugoocoto/chacha2
