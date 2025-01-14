# Setting up the environment

```ocaml
# #require "eio_main";;
```

```ocaml
open Eio.Std

module S = Eio.Stream

exception Cancel

let run fn =
  Eio_main.run @@ fun _ ->
  fn ()

let add ?sw t v =
  traceln "Adding %d to stream" v;
  S.add ?sw t v;
  traceln "Added %d to stream" v

let take ?sw t =
  traceln "Reading from stream";
  traceln "Got %d from stream" (S.take ?sw t)
```

# Test cases

Simple non-blocking case

```ocaml
# run @@ fun () ->
  let t = S.create 2 in
  add t 1;
  add t 2;
  take t;
  take t
+Adding 1 to stream
+Added 1 to stream
+Adding 2 to stream
+Added 2 to stream
+Reading from stream
+Got 1 from stream
+Reading from stream
+Got 2 from stream
- : unit = ()
```

Readers have to wait when the stream is empty:

```ocaml
# run @@ fun () ->
  Switch.top @@ fun sw ->
  let t = S.create 2 in
  add t 1;
  Fibre.both ~sw
    (fun () -> take t; take t)
    (fun () -> add t 2)
+Adding 1 to stream
+Added 1 to stream
+Reading from stream
+Got 1 from stream
+Reading from stream
+Adding 2 to stream
+Added 2 to stream
+Got 2 from stream
- : unit = ()
```

Writers have to wait when the stream is full:

```ocaml
# run @@ fun () ->
  Switch.top @@ fun sw ->
  let t = S.create 3 in
  add t 1;
  Fibre.both ~sw
    (fun () ->
      add t 2;
      add t 3;
      add t 4;
    )
    (fun () ->
      take t;
      take t;
      take t;
      take t
    )
+Adding 1 to stream
+Added 1 to stream
+Adding 2 to stream
+Added 2 to stream
+Adding 3 to stream
+Added 3 to stream
+Adding 4 to stream
+Reading from stream
+Got 1 from stream
+Reading from stream
+Got 2 from stream
+Reading from stream
+Got 3 from stream
+Reading from stream
+Got 4 from stream
+Added 4 to stream
- : unit = ()
```

A zero-length queue is synchronous:

```ocaml
# run @@ fun () ->
  Switch.top @@ fun sw ->
  let t = S.create 0 in
  Fibre.both ~sw
    (fun () ->
      add t 1;
      add t 2;
    )
    (fun () ->
      take t;
      take t;
    )
+Adding 1 to stream
+Reading from stream
+Got 1 from stream
+Reading from stream
+Added 1 to stream
+Adding 2 to stream
+Added 2 to stream
+Got 2 from stream
- : unit = ()
```

Cancel reading from a stream:

```ocaml
# run @@ fun () ->
  let t = S.create 1 in
  try
    Switch.top @@ fun sw ->
    Fibre.both ~sw
      (fun () -> take ~sw t)
      (fun () -> Switch.turn_off sw Cancel);
    assert false;
  with Cancel ->
    traceln "Cancelled";
    add t 2;
    take t
+Reading from stream
+Cancelled
+Adding 2 to stream
+Added 2 to stream
+Reading from stream
+Got 2 from stream
- : unit = ()
```

Cancel writing to a stream:

```ocaml
# run @@ fun () ->
  let t = S.create 1 in
  try
    Switch.top @@ fun sw ->
    Fibre.both ~sw
      (fun () -> add ~sw t 1; add ~sw t 2)
      (fun () -> Switch.turn_off sw Cancel);
    assert false;
  with Cancel ->
    traceln "Cancelled";
    take t;
    add t 3;
    take t
+Adding 1 to stream
+Added 1 to stream
+Adding 2 to stream
+Cancelled
+Reading from stream
+Got 1 from stream
+Adding 3 to stream
+Added 3 to stream
+Reading from stream
+Got 3 from stream
- : unit = ()
```

Cancel writing to a zero-length stream:

```ocaml
# run @@ fun () ->
  let t = S.create 0 in
  try
    Switch.top @@ fun sw ->
    Fibre.both ~sw
      (fun () -> add ~sw t 1)
      (fun () -> Switch.turn_off sw Cancel);
    assert false;
  with Cancel ->
    traceln "Cancelled";
    Switch.top @@ fun sw ->
    Fibre.both ~sw
      (fun () -> add ~sw t 2)
      (fun () -> take ~sw t)
+Adding 1 to stream
+Cancelled
+Adding 2 to stream
+Reading from stream
+Got 2 from stream
+Added 2 to stream
- : unit = ()
```

Trying to use a stream with a turned-off switch:

```ocaml
# run @@ fun () ->
  let t = S.create 0 in
  Switch.top @@ fun sw ->
  Switch.turn_off sw Cancel;
  begin try add  ~sw t 1 with ex -> traceln "%a" Fmt.exn ex end;
  begin try take ~sw t   with ex -> traceln "%a" Fmt.exn ex end
+Adding 1 to stream
+Cancelled: Cancel
+Reading from stream
+Cancelled: Cancel
Exception: Cancel.
```

Readers queue up:

```ocaml
# run @@ fun () ->
  let t = S.create 0 in
  Switch.top @@ fun sw ->
  Fibre.fork_ignore ~sw (fun () -> take t; traceln "a done");
  Fibre.fork_ignore ~sw (fun () -> take t; traceln "b done");
  Fibre.fork_ignore ~sw (fun () -> take t; traceln "c done");
  add t 1;
  add t 2;
  add t 3
+Reading from stream
+Reading from stream
+Reading from stream
+Adding 1 to stream
+Added 1 to stream
+Adding 2 to stream
+Added 2 to stream
+Adding 3 to stream
+Added 3 to stream
+Got 1 from stream
+a done
+Got 2 from stream
+b done
+Got 3 from stream
+c done
- : unit = ()
```

Writers queue up:

```ocaml
# run @@ fun () ->
  let t = S.create 0 in
  Switch.top @@ fun sw ->
  Fibre.fork_ignore ~sw (fun () -> add t 1);
  Fibre.fork_ignore ~sw (fun () -> add t 2);
  Fibre.fork_ignore ~sw (fun () -> add t 3);
  take t;
  take t;
  take t
+Adding 1 to stream
+Adding 2 to stream
+Adding 3 to stream
+Reading from stream
+Got 1 from stream
+Reading from stream
+Got 2 from stream
+Reading from stream
+Got 3 from stream
+Added 1 to stream
+Added 2 to stream
+Added 3 to stream
- : unit = ()
```
