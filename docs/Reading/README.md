# Pipelines Part 3 - Readers

**NEEDS WORK** - this text was migrated from a proposed blog article and needs restructuring to be suitable as a docs item, and updating to the current API

---

In part 1 we introduced the general aims of pipelines and discussed the abstract `MemoryPool<byte>` (and the default `MemoryPool` implementation of that). In part 2 we looked at how to create a `Pipe`, and how to write data into the pipe via the `IPipeWriter` API. In this part, we're going to look at how to *consume* data from a pipe.

A `Pipe` exposes an `IPipeReader` in virtually the same way that it exposes the `IPipeWriter` that we looked at previously; just like before, we have two identical options:

    IPipeReader reader = pipe.Reader; // option 1
    IPipeReader reader = pipe; // option 2

The API for *reading* a pipe is in many ways quite different to the API for *writing*, although it still builds upon the block-based approach that comes from the memory-pool.

So; let's get hold of a reader and try reading; firstly, note that because data doesn't arrive usually all at once, we almost always want to *loop* while reading:

```
while (true)
{
    ReadResult readResult = await reader.ReadAsync();
    ReadOnlyBuffer buffer = readResult.Buffer;
    // TODO
}
```

This shows us looping over the input (asynchronously as data becomes available), and obtaining the `ReadResult` and `ReadOnlyBuffer` each time. The `ReadResult` describes the state of the read operation (including telling us whether the writer has been completed, i.e. is there ever going to be any more data?); the `ReadOnlyBuffer` allows us to talk about **multiple chunks** (typically multiple blocks from the memory pool) of data at a single time.

A huge difference between a `Pipe` and a `Stream` is that when you `Read` from a `Stream`, you have consumed the data: it is now your responsibility, your problem. This is massively inconvenient in many cases - especially with "framed" data protocols. When reading from a `Stream` you a very rarely lucky enough to end up reading **exactly** one logical "frame" of data. Usually you end up reading half of a frame, or 2 entire frames and half of the next frame. And now you need to do something with the data that isn't yet usable, either because you don't have enough to do anything *at all*, or because you managed to do something but still had some data left over. This is the "back buffer" problem that I mentioned in part 1.

Pipelines solves this by having `ReadAsync` allow you to look at the data *without* considering it "consumed". Instead, you have the opportunity to look at the data, and then *tell the pipe* how much of it you were able to use. That might mean none of it (because you don't have an entire frame), or it might mean all of it, or it might mean some of it. Data we mark as consumed is released back to the memory pool for re-use. Data we *don't* consume will be retained by the pipe, and *given to us again next time*. Additionally, you can not only tell it how much you *consumed*, but you can also tell it how much you *looked at*:

- if you *didn't look at* all the data, then when you next `ReadAsync`, it can immediately return control to you, for you to have another look at the data you didn't *actively consume*
- if you *did* look at all the data, it knows that there isn't much point in waking you up again until more data becomes available
- if it can see that you're not looking at all the data but also not consuming anything, then it can interrupt you to prevent you looping forever without advancing
- if it can see that you looked at all the data and the writer is completed (no more data will ever arrive), it can determine that you've got as far as you ever will

We achieve this via the `Advance()` method on the reader; we *must* tell it what we consumed, and we can *optionally* tell it what we looked at.

As a very simple example, we might choose to buffer everything before processing it (note: I'm not suggesting this is a good thing to do) - essentially `ReadToEnd()`. To do that, we can use:

```
while (true)
{
    ReadResult readResult = await reader.ReadAsync();
    ReadOnlyBuffer buffer = readResult.Buffer;
    if (!readResult.IsCompleted)
    {
        // keep pushing it back; buffer it until the end
        reader.Advance(buffer.Start, buffer.End);
    }
    else
    {
        // TODO
    }
}
```

The `buffer.Start` and `buffer.End` properties are `Position` values - essentially cursors in a sequence of blocks. We can use those `Position` values in our call to `Advance` - the first parameter (`buffer.Start`) tells it "we didn't consume anything past the start"; the second parameter (`buffer.End`) tells it "assume we've looked at it all - don't nudge me again until either more data arrives or the writer is completed".

So; once everything is completed, what can we do? `ReadOnlyBuffer` looks and feels a lot like `Memory<byte>` did. It has a `.Length`, and we can "slice" it to create sub-regions (which is also `ReadOnlyBuffer`). The naming is interesting: *read only*. As we explore this API, we'll see that where we had `Memory<T>` and `Span<T>`, we now have `ReadOnlyMemory<T>` and `ReadOnlySpan<T>`. You won't be amazed to hear that this lacks the `set` indexer; this no longer works:

    span[0] = 0x10; // nope!

Because we are *reading* data, it is not expected that we should usually want to mutate it. This isn't a strong "this will never change" guarantee - unless you use dedicated hardware, read-only just isn't really "a thing" in computing, and *if you need to*, you can force the read-only versions into their mutable cousins. But as long as everyone follows the rules, this is fine. Actually, this is nothing new; people rarely worry about this:

    string s = ...
    CallSomeMethod(s);

because they know that string is immutable so `s` will contain the same value when it comes back. Here's the kicker: `string` *isn't really immutable*, and code *can* change the value if it plays dirty. Think of `ReadOnlyMemory<T>` in the same way as `string`.


We previously noted that `Memory<byte>` was *indirect* access to memory; well, `ReadOnlyBuffer` is *doubly* indirect - and wraps multiple `ReadOnlyMemory<byte>` chunks. A lazy way to process this data would be to linearize it via `ToArray()`:

```
while (true)
{
    ReadResult readResult = await reader.ReadAsync();
    ReadOnlyBuffer buffer = readResult.Buffer;
    if (!readResult.IsCompleted)
    {
        // keep pushing it back; buffer it until the end
        reader.Advance(buffer.Start, buffer.End);
    }
    else
    {
        byte[] arr = buffer.ToArray();
        string message = Encoding.UTF8.GetString(arr);
        Console.WriteLine(message);
        // tell it that we consumed all the data
        reader.Advance(buffer.End);
        break; // we've finished!
    }
}
```

Once again, this *will work*, but represents a failure to properly use the pipelines API: we've forced an additional transient buffer to be created. So what can we do?

Let's assume we want to write a helper extension method (note: our caller will probably be `async`, and we will *probably* want to work with `Span<T>`, so our helper method must be synchronous):

```
public static unsafe string ReadUtf8(
        ref this ReadOnlyBuffer buffer)
{
    if (buffer.IsEmpty) return "";
    // TODO
}
```

allowing our calling code to become:

```
string message = buffer.ReadUtf8();
Console.WriteLine(message);
```

If we're lucky, we'll find that we only *actually* have a single range to consider, which makes a lot of things simpler. For example, we can get a `string` *relatively* easily if we only have a single range, remembering that *depending on what APIs support `Span<T>`*, we might need to use `unsafe` code and get back to an unmanaged pointer:

```
if (buffer.IsSingleSpan)
{
    var span = buffer.First.Span;
    fixed (byte* bytes = &MemoryMarshal.GetReference(span))
    {
        return Encoding.UTF8.GetString(bytes, span.Length);
    }
}
```

Since we're going to want to consume *all* the data in the chunks in one go, we can make use of the fact that `ReadOnlyBuffer` supports `foreach` usage, providing a sequence of `Memory<byte>`, allowing the worst-case multi-buffer version of our method to become:

```
// let's assume that it is small enough to decode
// on the stack; each byte can be *at most* one
// character
int charsSpace = checked((int)buffer.Length);
char* chars = stackalloc char[charsSpace];

var decoder = Encoding.UTF8.GetDecoder();
int charsWritten = 0;

foreach(var chunk in buffer)
{
    var span = chunk.Span;
    fixed (byte* bytes = &MemoryMarshal.GetReference(span))
    {
        int charsAdded = decoder.GetChars(
            bytes, span.Length,
            chars + charsWritten,
            charsSpace, flush: false);

        charsWritten += charsAdded;
        charsSpace -= charsAdded;
    }
}
return new string(chars, 0, charsWritten);
```

More commonly, we want to look at the data in pieces - not all at once, essentially iterating over the various `Span<byte>` that make the multiple chunks in the `ReadOnlyBuffer` consuming only a few bytes at a time. A typical example might be a length-prefixed binary protocol - for example, we could have a series of text commands sent as:

- 4 bytes, little-endian int32 = *payload length*
- *payload length* bytes of utf-8 text

That would be a pretty common scenario, especially if individual commands might need to include things like CRLF. So we might want to write a server as, conceptually:

```
while (running)
{
    await data
    if (have < 4 bytes) push back and try again later;
    len = parse 4 bytes
    if (have < len bytes) push back and try again later;
    message = decode len bytes as utf8
    process message
    record 4 + len bytes as consumed
}
```

We *could* do this by using various methods on the `ReadOnlyBuffer`, including `Slice()` - and like we saw with our previous example: either accessing the `.First` range, or using `foreach` to iterate over the ranges. But this is *not* usually a good approach when processing small amounts of data from `ReadOnlyBuffer`; all of the slice work, the position work, and accessing spans from memories: is *relatively* expensive. And also really quite inconvenient. We can make it more efficient *and* easier to use ***at the same time*** by making use of  `BufferReader` (`BufferReader<ReadOnlyBuffer>` now - TBC).

A `BufferReader` is a `ref struct` - so stack-only; it *directly* holds the current `ReadOnlySpan<T>` as a field (something that only a `ref struct` can do), avoiding the "span fetch" cost associated with repeatedly accessing `ReadOnlyMemory<T>`. It also avoids constantly slicing through the span, instead advising the consumer how to track the intended index in a span that *doesn't change* until it needs to switch to the next block.

The main ways of fetching data from a `BufferReader` are:

- `Take()` consumes a single byte in one go (much like `Stream.ReadByte()`), returning a negative number if we have exhausted the data
- `.Span` and `.Index` provide access to the current block - you should start looking *at `.Index`, not at zero
- `.Skip(...)` moves forwards through the reader, which *might* mean changing `.Index` and staying in the same span (`.Span`), or it *might* mean moving to the *next* span and resetting the index
- finally, `ConsumedBytes` tells us how far we have read in total - it is essentially the sum of all the skips

We can see this by looking at how we might read the 4-byte length prefix efficiently:

```
public static int ReadLittleEndianInt32(
    ref this BufferReader reader)
{
    // are there 4 bytes left in the
    // current span?
    var span = reader.Span;
    if (span.Length >= reader.Index + 4)
    {
        int i = reader.Index;
        var val = span[i++]
            | (span[i++] << 8)
            | (span[i++] << 16)
            | (span[i] << 24);
        reader.Skip(4);
        return val;
    }
    // read 4 bytes individually (may cross spans)
    int a = reader.Take(), b = reader.Take(),
        c = reader.Take(), d = reader.Take();
    // cheeky MSB check
    if ((a | b | c | d) < 0)
        throw new EndOfStreamException();
    return a
        | (b << 8)
        | (c << 16)
        | (d << 24);
}
```

Once again we've gone for a "fast path" when we have enough data in the single span, reading 4 successive bytes, then using `Skip()` to logically consume them. Conversely, if we need to use multiple spans (which should be rare), we might as well simply read 4 bytes and then compose them. Since we're pre-checking that we expect 4 bytes in the buffer, it isn't a problem to throw the `EndOfStream` exception if something goes wrong, but we could also have used a `Try*` pattern if we prefer not to throw.

I'm *not* going to re-implement `ReadUtf8` against the `BufferReader`, since we've already covered utf-8 decoding more than enough - the only significant difference would be that in the worst-case code path (multiple spans), instead of a `foreach` loop we'd have a `while` loop, and (very importantly) we need to remember to take `reader.Index` into account against each span, something like:

```
while(bytes > 0 && !reader.End)
{
    var span = reader.Span;
    fixed (byte* bytes = &MemoryMarshal.GetReference(span))
    {
        int bytesThisSpan = Math.Min(
            bytes, span.Length - reader.Index);

        int charsAdded = decoder.GetChars(
            bytes + reader.Index, bytesThisSpan,
            chars + charsWritten,
            charsSpace, flush: false);
        // ...
    }
} 
if (bytes != 0) throw new EndOfStreamException();
```

So, with the above, let's assume we also have a `ReadUtf8` extension method against `BufferReader` that takes a number of bytes. Now we can write our server method:

```
public string TryReadMessage(in ReadOnlyBuffer buffer)
{
    long bufferLength = buffer.Length;

    if (bufferLength < 4) return null;

    var reader = BufferReader(buffer);
    var msgLength = reader.ReadLittleEndianInt32();

    if (bufferLength < msgLength + 4) return null;

    var msg = reader.ReadUtf8(msgLength);
    buffer = buffer.Slice(reader.ConsumedBytes);
    return msg;
}
```

with our server loop now as simple as:

```
while(true)
{
    var result = await reader.ReadAsync();
    var buffer = result.Buffer;
    if (result.IsCompleted && buffer.IsEmpty)
        break;

    var msg = TryReadMessage(buffer);
    if (msg == null)
    {
        // if we're returning null, we didn't have
        // enough bytes, so consider it fully
        // inspected
        reader.Advance(buffer.Start, buffer.End);
    }
    else
    {
        // note we haven't inspected anything past
        // the message, so don't claim to have
        reader.Advance(buffer.Start);
        ProcessMessage(msg);
    }
}
```

So, we've explored how to use the `IPipeReader` API, using `ReadOnlyBuffer` and `BufferReader` - and how to `Advance()` through the pipe. Next up, we'll look at how we can actually *do things* with pipes - how we can plumb systems together using pipelines.