+++
date = '2025-03-19T19:44:50-07:00'
draft = true
title = 'Some Nifty MacOS Terminal Commands'
[params]
    rank = 10
+++
In the course of making [Playtree](/posts/playtree-technical-overview), I came across some esoteric terminal commands for MacOS. To start with the least useful:
```
afplay <audio_file>
```
will play a specified audio file from the command line. I used this in an early CLI prototype for Playtree, but its utility is pretty limited. There's no way to control audio once playback starts. You can stop playback by sending an interrupt signal with `CTRL-C`, and you can "mute" audio by suspending the process with `CTRL-Z`. This doesn't pause playback, though. It's still playing through silently in the background.

There are few command line options, though. You can set the volume and the time to play before ending playback.You can set the playback rate, too. It's a fun old command buried in the OS. There's potential, here, to pull a prank on a friend, if only people still had audio files on hand. Maybe the `say` command, which reads the input text aloud, would suffice.

Make sure to check out the `afinfo` and `afconvert` commands, which I haven't tried but might actually be useful!

---

You probably already use the next pair of commands all the time, just not on the command line. They are `pbcopy` and `pbpaste`, available on MacOS.

`pbcopy` copies `stdin` to the clipboard, and `pbpaste` outputs the clipboard to `stdout`. For example:

```
echo 'This is a sentence' | pbcopy
```
will copy `This is a sentence` to the clipboard. This can be useful if you need to copy the contents of a file, for instance:
```
cat file.txt | pbcopy
```

Similarly, you can paste the clipboard to a file:
```
pbpaste > new_file.txt
```
```
pbpaste >> file_to_append_to.txt
```

I found these commands pretty useful when I needed to interface between my browser and my terminal. The easiest way to get text from the browser is to copy it, and often the easiest way to interact with the local filesystem is via the command line.

Another feature of `pbcopy` and `pbpaste` is that you can target one of four clipboards with the `-pboard` flag. I didn't end up using this feature very often, since I was almost always using these commands in conjunction with normal copy and paste, but maybe you can make good use of it.

---

The last tidbit is a misnomer on two counts—it isn't exclusive to MacOS, and the takeaway isn't about the command itself! The command is `less`. You can use `less` to make terminal output more human readable. It'll take an input stream and display only a fraction of the stream at a time. You can move within the stream using the arrow keys and by entering commands.

You may already know about `less`. What you may not know is that `less` outputs the stream it's reading to `stdout`! This means that you can do something like
```
echo "This is some text to read" | less | cat
```
and, instead of seeing the text in `less`, it will just be printed to the terminal.

Why would you ever want to do that? Isn't it redundant? The main benefit is that *other* commands may utilize `less` to display their output. `git diff` works this way. While working on Playtree, I'd often pepper the code with `debugger` and `console.log` statements. Before committing my code, I'd check the diff to see if I'd left any stragglers around. I realized you can just pipe `git diff` to `grep`, and it was much faster than sifting through the diff in `less`:
```
git diff | grep console.log
```

You can even pipe the output of `grep` *back* to `less`:
```
git diff | grep console.log | less
```

This made it much easier to sift through diffs if I needed to. Granted, there's a lot I've yet to learn about how to move around a file quickly in `less` itself. I'll probably get to that when I get around to vim.

---

That's all I've got for now. Check out [this reference](https://ss64.com/mac/) for other available MacOS commands. This index is where I found out about the ones I shared with you.
