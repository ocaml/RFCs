# modular IO channels in the stdlib

This RFC proposes to update the stdlib's types `in_channel` and `out_channel` to make them user-definable and composable.

Even though a lot of networked applications use lwt or async to achieve high levels of concurrency, classic blocking IO still has its uses. However, the standard OCaml channels suffer from some annoying limitations:

- they cannot be created outside the stdlib, which means the only thing we can manipulate through them is sockets and other unix file descriptors.
- they cannot be composed. In other languages [such as Go](https://golang.org/pkg/io/#Reader), one can write reader or writer combinators which transform the bytestream
that is written or read. Typical examples would include (de)compression and encryption.
- there is some duplication in `Printf`; namely, the existence, and incompatibility, of `bprintf`, `sprintf`, and `fprintf`. This makes `Printf` printers artificially limited
  since their type determines what kind of output they can produce. In my own experience, `Format` is a better choice because a single `Format.formatter -> t -> unit` function
  can be used in more cases than any single `Printf` function, despite the overhead of formatters.

As a consequence, many libraries have their own opaque channel types that
are not compatible with the standard ones. That's a missed opportunity for
code reuse and composability.

## Extensibility

The current types are implemented in C and are opaque. I propose that it be changed for:

```ocaml
type in_channel =
  | IC_raw of old_in_channel (* implemented in C *)
  | IC_user of {
    read: bytes -> int -> int -> int;
    close: unit -> unit;
  }

type out_channel =
  | OC_raw of old_out_channel (* implemented in C *)
  | OC_user of {
    write: bytes -> int -> int -> int;
    flush: unit -> unit;
    close: unit -> unit;
  }

(* now doable in userland: *)

val ic_of_string : string -> in_channel
val unzip : in_channel -> in_channel

val oc_of_buf : Buffer.t -> out_channel
val zip : out_channel -> out_channel
val encrypt_rot13: out_channel -> out_channel

(* Write to both channels *)
val tee : out_channel -> out_channel -> out_channel

```

In fact, I'm not sure why the channels are still implemented in C. I think the base case could be
a raw Unix file descriptor and a `bytes` buffer — but maybe this is needed for portability.

## Interface improvement for `in_channel`

The current interface of `in_channel` provides, roughly, `input : bytes -> int -> int -> int`
which takes a byte slice and returns how many bytes were read, `0` indicating end of input.
This interface doesn't expose the underlying buffer  and instead imitates the lower level posix APIs.

The problem of this interface is that it makes some functions quite awkward to write.
An interesting alternative is [rust's `BufRead` interface](https://doc.rust-lang.org/std/io/trait.BufRead.html).
In OCaml, that corresponds roughly to:

```ocaml
module In_channel : sig
  type t

  (** Obtain a slice of the current buffer. Empty iff EOF was reached *)
  val fill_buf : t -> (bytes * int * int)

  (** Consume n bytes from the input. *)
  val consume : t -> int -> unit

  (** Close channel and release resources *)
  val close : t -> unit
end
```

The semantics of these operations is:

- `fill_buf ic` ensures that the channel's internal buffer is non-empty, unless
  end-of-input was reached. Then, it exposes a view of the internal buffer.
  The important aspect of this is that successive calls to `fill_buf` return
  the same result; this doesn't consume input on a logical level. This function
  just exposes a slice of the input.
- `consume ic n` eats `n` bytes of the input. It must only be called after
  `fill_buf` returned a slice of length at least `n`. If the whole slice
  exposed by `fill_buf` was consumed, then the channel will have to read more
  from its underlying stream at the next call to `fill_buf`.
- `close ic` closes the channel and releases underlying resources.

### Advantages

This interface is easier to use than the current `input` interface, especially when
parsing formats with non-trivial framing (e.g. http1.1). One typically wants
to read a line to get headers and framing (content-length) information, followed
by a read of `n` bytes. It is therefore important to read the line(s) efficiently
but without consuming _too much_ from the input buffer as it's possibly part
of the payload.

Compare the stdlib's [`input_line` implementation](https://github.com/ocaml/ocaml/blob/f333db8b0f176b1d75e6fdb46a97a78995426ed7/stdlib/stdlib.ml#L439)
which uses a magical external to look inside the C buffer, with this snippet
(adapted from [tiny httpd](https://github.com/c-cube/tiny_httpd/blob/3ac5510e2d5dfcdf448a03a99c0c178b73afeabd/src/Tiny_httpd.ml#L159)):

```ocaml
let input_line (ic:In_channel.t) : String.t =
  let buf = Buffer.create 32 in
  let continue = ref true in
  while !continue do
    let s, i, len = In_channel.fill_buf ic in
    if len=0 then (
      continue := false;
      if Buffer.length buf = 0 then raise End_of_file;
    );
    let j = ref i in
    (* look for ['\n'] in the input buffer *)
    while !j < i+len && Bytes.get s !j <> '\n' do
      incr j
    done;
    if !j-i < len then (
      assert (Bytes.get s !j = '\n');
      Buffer.add_bytes buf s i (!j-i); (* without '\n' *)
      In_channel.consume ic (!j-i+1); (* consume rest of line + '\n' *)
      continue := false
    ) else (
      Buffer.add_bytes buf s i len;
      In_channel consume ic len;
    )
  done;
  Buffer.contents buf
```

### Compatibility

The current type of channels could retain its interface, for retro-compatibility,
in addition to the new interface which exposes `consume` and `fill_buf`,
but implement `input`, in the general case, as follows
(adapted from [tiny httpd](https://github.com/c-cube/tiny_httpd/blob/3ac5510e2d5dfcdf448a03a99c0c178b73afeabd/src/Tiny_httpd.ml#L146)):

```ocaml
let input (ic:In_channel.t) buf i len : int =
  let offset = ref 0 in
  let continue = ref true in
  while continue && !offset < len do
    let s, j, n = In_channel.fill_buf ic () in
    let n_read = min n (len - !offset) in
    Bytes.blit s j buf (i + !offset) n_read;
    offset := !offset + n_read;
    In_channel.consume ic n_read;
    if n_read=0 then continue := false; (* eof *)
  done;
  !offset

```

In most cases this should only do one iteration if `n` is smaller than the
underlying buffer's size.

**Alternatively**, this can be considered the implementation of `really_input`,
and have input be just:

```ocaml
let input (ic:In_channel.t) buf i len : int =
  let s, j, n = In_channel.fill_buf ic in
  let n_read = min n len in
  Bytes.blit s j buf i n_read;
  In_channel.consume ic n_read;
  n_read
```

Here we see that the classic `input` is simply the successive application
of `fill_buf` and `consume`.
