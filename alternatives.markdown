So if ruby 1.9 got it wrong, how could it have gotten it right?

Have a look at how Python 3 handles strings and encodings:
http://docs.python.org/release/3.0.1/whatsnew/3.0.html#text-vs-data-instead-of-unicode-vs-8-bit

Python 3 has two completely distinct classes: *str* (a sequence of unicode
*characters*), and *bytes* (a sequence of bytes).  Objects of the two
classes cannot be combined.  A statement like

    a = b + c

will always fail at runtime if b is str and c is bytes, or vice versa. This
supports the basic principle of "fail early, fail often": that is, if I did
something wrong in my program, I want it to fail immediately and
consistently.

The Python documentation says:

> "As the str and bytes types cannot be mixed, you must always explicitly
> convert between them.  Use str.encode() to go from str to bytes, and
> bytes.decode() to go from bytes to str.  You can also use bytes(s,
> encoding=...) and str(b, encoding=...), respectively."

Compare this with the muddle in ruby 1.9. A string marked with encoding
"UTF-8" may successfully be combined with a string marked "BINARY",
sometimes.  This is data-dependent: under some combinations of input data,
it will crash.  However, even if it doesn't crash, under some circumstances
the result string will be marked "UTF-8" and in other circumstances the
result string will be marked "BINARY".  And that, in turn, affects whether
later uses of this value will succeed or crash.  Program analysis becomes a
nightmare.  If you are unlucky, you can write a bad program but it won't
crash until much later, when someone feeds it some different data.

Python forces you to be explicit about conversions. When you open a file for
reading, you state whether you want to get the raw bytes, or whether to
decode those bytes into characters.  If you want characters (str), then you
specify the encoding - or have it default to using the locale, as ruby 1.9
does.  However, the choice you have made between binary data and characters
is clearly marked in the class of the data as you subsequently process it.

Similarly, if you have a character *str* and you want to write it out, this
involves convering it into an encoded representation, so you specify what
output encoding to use.  This is symmetrical: `bytes -> str -> bytes`.  ruby
1.9 is asymmetrical; the default external encoding for a file opened for
write is nil, which means whatever bytes the String happens to contain,
regardless of its tagged encoding, get written to the file.

So: my basic proposal is that Ruby 1.9 could have had class String and class
Bytes, which are completely incompatible with each other.  String#length
would give you the number of characters; Bytes#length would give you the
number of bytes.  String#bytesize would be meaningless, because characters
themselves don't have any bytesize - only characters *which have been
encoded as bytes using some encoding*.  There would need to be separate
literal forms for String and Bytes, like Python.

Now, one limitation of Python 3's approach is that it only handles text in
the unicode character set.  Almost every commonly-used encoding encodes a
subset of unicode.  However there are a few Asian character sets which
contain characters outside of the unicode set, such as GB2312.  Well, that's
no problem: you could just have a separate class for each character set you
wish to support, such as a GB2312String.

In that case, any attempt to combine two different classes, such as a String
with a GB2312String, would fail.  If you really want to combine them, you'd
have to transcode explicitly - at which point you'd say what to do about
characters which don't have an equivalent in the other character set, and
which character set you're transcoding to.  If you forget to do this, your
program would fail immediately and in an obvious way.
