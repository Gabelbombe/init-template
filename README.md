### How to utilize dynamic initd script

A simple template for init scripts that provide the start, stop,
restart and status commands.

Handy for [Node.js](http://http://nodejs.org/) apps and everything
else that runs itself.

Copy _template_ to /etc/init.d and rename it to something
meaningful. Then edit the script and enter that name after _Provides:_
(between _### BEGIN INIT INFO_ and _### END INIT INFO_).

Now set the following three variables in the script:

##### dir

The working directory of your process.

##### cmd

The command line to start the process.

##### user

The user that should execute the command (optional).
If not set, the command will be called as root (via `sudo ...`).


```bash
dir=""        ## /usr/local/bin
cmd=""        ## foo-1.2 {ARGS}
cnf=""        ## /etc/foo.conf
user=""       ## bar
lopts=""      ## --config
```

#### Usage

##### Start

Starts the app.

    /etc/init.d/algorithms start

##### Stop

Stops the app.

    /etc/init.d/algorithms stop

##### Restart

Restarts the app.

    /etc/init.d/algorithms restart

##### Status

Tells you whether the app is running. Exits with _0_ if it is and _1_
otherwise.

    ``/etc/init.d/algorithms status`

Logging
-------

By default, standard output goes to _/var/log/scriptname.log_ and
error output to _/var/log/scriptname.err_. If you're not happy with
that, change the variables `stdout_log` and `stderr_log`.


### Adding to upstart onboot

```bash
## To add as service `foo` start
`bash# chkconfig --add foo`


## On boot
cd etc rc3.d ; ln -s ../init.d/foo foo
```

### Dynamic lopts flagging

So... about `getopt`, the `getopts` builtin can be used to handle only long options like this:

```bash
while getopts :-: o
do  case "$o$OPTARG" in
(-longopt1) process ;;
(-longopt2) process ;;
esac; done
```

Of course, as is, that doesn't work if the long-options are supposed to have arguments. It can be done, though, but, as I've learned working on this. While I initially included it here I realized that for long-options it doesn't have much utility. In this case it was only shortening my `case (match)` fields by a single, predictable character. Now, what I know, is that it is excellent for short options - it is most useful when it is looping over a string of unknown length and selecting single bytes according to its option-string. But when the option is the arg, there's not a lot you're doing with a `for var do case $var in` combination that it could do. It is better, I think, to keep it simple.

I suspect the same is true of `getopt` but I don't know enough about it to say with any certainty. Given the following arg array, I will demonstrate my own little arg parser - which depends primarily on the evalation/assignment relationship I've come to appreciate for `alias` and `$((shell=math))`.

```bash
set -- this is ignored by default --lopt1 -s 'some '\''
args' here --ignored   and these are ignored \
--alsoignored andthis --lopt2 'and

some "`more' --lopt1 and just a few more
```

That's the arg string I'll be working with. Now:

```bash
aopts() { env - sh -s -- "$@"
} <<OPTCASE 3<<\OPTSCRIPT
acase() case "\$a" in $(fmt='
        (%s) f=%s; aset "?$(($f)):";;\n'
        for a do case "$a" in (--) break;;
        (--*[!_[:alnum:]]*) continue;;
        (--*) printf "$fmt" "$a" "${a#--}";;
        esac;done;printf "$fmt" '--*' ignored)
        (*) aset "" "\$a";;esac
shift "$((SHIFT$$))"; f=ignored; exec <&3
OPTCASE
aset()  {  alias "$f=$(($f${1:-=$(($f))+}1))"
        [ -n "${2+?}" ] && alias "${f}_$(($f))=$2"; }
for a do acase; done; alias
#END
OPTSCRIPT
```

That processes the arg array in one of two different ways depending on whether you hand it one or two sets of arguments separated by the `--` delimiter. In both cases it applies to sequences of processing to the arg array.

If you call it like:

```bash
: $((SHIFT$$=3)); aopts --lopt1 --lopt2 -- "$@"
```

Its first order of business will be to write its `acase()` function to look like:

```bash
acase() case "$a" in
    (--lopt1) f=lopt1; aset "?$(($f)):";;
    (--lopt2) f=lopt2; aset "?$(($f)):";;
    (--*) f=ignored; aset "?$(($f)):";;
    (*) aset "" "$a";;esac
```

And next to shift 3. The command-substitution in the acase() function definition is evaluated when the calling shell builds the function's input here-documents, but acase() is never called or defined in the calling shell. It is called in the subshell, though, of course, and so this way you can dynamically specify the options of interest on the command line.

If you hand it an un-delimited array it simply populates `acase()` with matches for all arguments beginning with the string `--`.

The function does practically all of its processing in the subshell - iteratively saving each of the arg's values to aliases assigned with associative names. When it is through it prints out every value it saved with alias - which is POSIX-specified to print all saved values quoted in such a way that their values can be reinput to the shell. So when I do...

```bash
aopts --lopt1 --lopt2 -- "$@"
```

Its output looks like this:

```bash
...ignored...
lopt1='8'
lopt1_1='-s'
lopt1_2='some '\'' args'
lopt1_3='here'
lopt1_4='and'
lopt1_5='just'
lopt1_6='a'
lopt1_7='few'
lopt1_8='more'
lopt2='1'
lopt2_1='and
```

As it walks through the arg list it checks against the case block for a match. If it finds a match there it throws a flag - `f=optname`. Until it once again finds a valid option it will add each subsequent arg to an array it builds based on the current flag. If the same option is specified multiple times the results compound and do not override. Anything not in case - or any arguments following ignored options - are assigned to an _ignored array_.

The output is shell-safed for shell-input automatically by the shell, and so:

```bash
eval "$(: $((SHIFT$$=3));aopts --lopt1 --lopt2 -- "$@")"
```

...should be perfectly safe. If it for any reason is _not safe_, then you should probably file a bug report with your shell maintainer.

It assigns two kinds of `alias` values for each match. First, it sets a flag - this occurs whether or not an option precedes non-matching arguments. So any occurrence of `--flag` in the arg list will trigger `flag=1`. This does not compound - `--flag --flag --flag` just gets `flag=1`. This value _does_ increment though - for any arguments that might follow it. It can be used as an index key. After doing the `eval` above I can do:

```bash
printf %s\\n "$lopt1" "$lopt2"
```

...to get...

```bash
8
1
```

And so:

```bash
for o in lopt1 lopt2 ; do
  list= i=0
  echo "$o = $(($o))"
  while [ "$((i=$i+1))" -le "$(($o))" ] ; do
    list="$list $o $i \"\${${o}_$i}\" "
  done
 eval "printf '%s[%02d] = %s\n' $list"
done
```

OUTPUT
```bash
lopt1 = 8
lopt1[01] = -s
lopt1[02] = some ' args
lopt1[03] = here
lopt1[04] = and
lopt1[05] = just
lopt1[06] = a
lopt1[07] = few
lopt1[08] = more
lopt2 = 1
lopt2[01] = and

```

And to args which did not match I would substitute ignored in the above for ... in field to get:

```bash
ignored = 10
ignored[01] = this
ignored[02] = is
ignored[03] = ignored
ignored[04] = by
ignored[05] = default
ignored[06] = and
ignored[07] = these
ignored[08] = are
ignored[09] = ignored
ignored[10] = andthis
```
