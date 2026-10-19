
# 2. Shell, Files & Text Processing

## A. File & Directory Management

```bash
touch file.txt                 # create empty file / update timestamp
mkdir mydir                    # create directory
mkdir -p a/b/c                 # create nested directories
cp src.txt dest.txt            # copy file
cp -r srcdir destdir           # copy directory recursively
cp -a srcdir destdir           # archive copy (preserve everything)
mv old.txt new.txt             # rename or move
mv file.txt /tmp/              # move into a directory
rm file.txt                    # delete file
rm -r mydir                    # delete directory recursively
rm -rf mydir                   # force delete, no prompts (DANGEROUS)
rmdir emptydir                 # remove an empty directory only
ln -s /path/target linkname    # symbolic (soft) link
ln target hardlink             # hard link
rename 's/\.txt/\.md/' *.txt   # batch rename (perl-rename)
basename /a/b/file.txt         # -> file.txt
dirname /a/b/file.txt          # -> /a/b
readlink -f linkname           # resolve a symlink to its real path
```

Symbolic link vs hard link: a **soft link** is a pointer to a path (breaks if the target is deleted); a **hard link** is another name for the same underlying data (inode) and survives deletion of the original name.

### Troubleshooting

**"rm: cannot remove: Permission denied"** — you don't own the file or its parent directory. Check with `ls -ld .` (write permission on the *directory* is what allows deletion, not on the file). Use `sudo` if it's system-owned.

**"Device or resource busy" when deleting/unmounting** — a process is using the file/dir. Find it with `lsof +D /path` or `fuser -vm /path`, then stop that process.

**"No space left on device" but `df` shows free space** — you've run out of **inodes**, not bytes. Check `df -i`; delete lots of small files (often in `/tmp`, mail spools, or session dirs).

**Accidentally deleted an important file** — there's no recycle bin. If it's still open by a process you may recover it via `/proc/PID/fd/`. Otherwise stop writing to that disk immediately and try `extundelete` or `testdisk`.

**Can't delete a file with a weird name (spaces, dashes, unicode)** — reference it by inode: `ls -i` to get the number, then `find . -inum NUMBER -delete`. A leading-dash name needs `rm -- -filename` or `rm ./-filename`.

**"Argument list too long"** — too many files for one command. Use `find . -name '*.log' -delete` or pipe through `xargs`.

---

## B. Viewing & Editing Files

```bash
cat file.txt                   # print whole file
tac file.txt                   # print in reverse (last line first)
less file.txt                  # scrollable pager (q to quit, / to search)
more file.txt                  # simpler pager
head file.txt                  # first 10 lines
head -n 20 file.txt            # first 20 lines
tail file.txt                  # last 10 lines
tail -n 50 file.txt            # last 50 lines
tail -f /var/log/syslog        # follow a file live (great for logs)
nl file.txt                    # number the lines
wc file.txt                    # lines, words, bytes
wc -l file.txt                 # count lines only
```

Editors:

```bash
nano file.txt                  # beginner-friendly editor (Ctrl+O save, Ctrl+X exit)
vim file.txt                   # powerful modal editor
```

Vim survival kit: press `i` to insert, `Esc` to leave insert mode, `:w` save, `:q` quit, `:wq` save+quit, `:q!` quit without saving, `/text` search, `dd` delete line, `yy` copy line, `p` paste, `u` undo.

---

## C. Text Processing & Searching

```bash
grep "pattern" file            # find lines matching pattern
grep -i "pattern" file         # case-insensitive
grep -r "pattern" /dir         # recursive search in a directory
grep -n "pattern" file         # show line numbers
grep -v "pattern" file         # invert: lines NOT matching
grep -E "foo|bar" file         # extended regex (alternation)
grep -c "pattern" file         # count matching lines

find /path -name "*.log"       # find files by name
find / -type d -name "conf*"   # find directories
find . -type f -size +100M     # files larger than 100 MB
find . -mtime -7               # modified in last 7 days
find . -name "*.tmp" -delete   # find and delete
find . -type f -exec chmod 644 {} \;   # run a command on each result

locate filename                # fast filename search (uses a DB)
updatedb                       # refresh the locate database

sort file.txt                  # sort lines
sort -n file.txt               # numeric sort
sort -r file.txt               # reverse sort
uniq file.txt                  # remove adjacent duplicate lines
sort file | uniq -c            # count occurrences of each line

cut -d',' -f1 file.csv         # cut column 1 (comma delimiter)
awk '{print $1, $3}' file      # print fields 1 and 3
awk -F',' '{print $2}' f.csv   # awk with comma delimiter
sed 's/old/new/g' file         # substitute text (stream editor)
sed -n '5,10p' file            # print lines 5 to 10
tr 'a-z' 'A-Z' < file          # translate characters (lowercase->upper)
diff file1 file2               # show differences between files
comm file1 file2               # compare sorted files line-by-line
```

`grep`, `sed`, and `awk` are the classic text-processing trio. Regular expressions power all three; learning basic regex (`^` start, `$` end, `.` any char, `*` repeat, `[...]` class) pays off across the whole system.

### Troubleshooting

**grep finds nothing but the text is clearly there** — special characters may need escaping, or the pattern is case-sensitive (add `-i`). If matching a literal string with symbols, use `grep -F` (fixed strings). For patterns with `+`, `?`, `|`, use `grep -E`.

**grep returns "binary file matches" with no lines** — force text output with `grep -a`, or you're searching a binary; add `--text`.

**`sed -i` didn't change the file** — the pattern didn't match (test first without `-i`), or on macOS `sed -i` needs an argument. Always test the substitution by running it *without* `-i` first to see the output.

**`find` prints "Permission denied" spam** — you're scanning dirs you can't read. Silence it with `find / -name x 2>/dev/null`, or run with `sudo`.

**`locate` can't find a file you just created** — its database is stale. Run `sudo updatedb` first; `find` searches live and won't have this issue.

**awk/cut splits on the wrong thing** — the default delimiter is whitespace for awk, tab for cut. Set it explicitly: `awk -F',' ...` or `cut -d',' ...`. Watch for multiple spaces vs single tabs.

---

## D. Redirection, Pipes & Wildcards

Every process has three standard streams: **stdin (0)**, **stdout (1)**, **stderr (2)**.

```bash
command > file                 # redirect stdout to file (overwrite)
command >> file                # append stdout to file
command 2> errors.txt          # redirect stderr
command > out.txt 2>&1         # redirect both stdout and stderr
command &> all.txt             # both, shorthand (bash)
command < input.txt            # take stdin from a file
command1 | command2            # pipe: output of one into another
command | tee file             # print to screen AND save to file
```

Wildcards (globbing) expand file names before the command runs:

```bash
*        # any number of characters      -> ls *.txt
?        # exactly one character         -> ls file?.txt
[abc]    # any one of a, b, c            -> ls file[123].txt
[a-z]    # a range                       -> ls [a-c]*
{a,b,c}  # brace expansion               -> touch file{1,2,3}.txt
```

Chaining operators:

```bash
cmd1 && cmd2        # run cmd2 only if cmd1 succeeds
cmd1 || cmd2        # run cmd2 only if cmd1 fails
cmd1 ; cmd2         # run both regardless
cmd &               # run in the background
```

Example combining everything: `grep -i error /var/log/syslog | sort | uniq -c | sort -rn | head` — find error lines, count each unique one, show the most frequent.

---