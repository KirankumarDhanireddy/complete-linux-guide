# 10. Archiving, Kernel & Hardware

## Archiving & Compression

**tar** bundles many files into one archive; compression tools (`gzip`, `bzip2`, `xz`, `zip`) shrink them.

```bash
# tar (the 'flags' spell out the action)
tar -cvf archive.tar files/        # create an (uncompressed) archive
tar -xvf archive.tar               # extract
tar -tvf archive.tar               # list contents without extracting
tar -czvf archive.tar.gz files/    # create gzip-compressed (.tar.gz)
tar -xzvf archive.tar.gz           # extract gzip
tar -cjvf archive.tar.bz2 files/   # bzip2 compression
tar -cJvf archive.tar.xz files/    # xz compression (best ratio)
tar -xvf archive.tar.gz -C /dest   # extract to a specific directory

# Flag meanings: c=create x=extract t=list v=verbose f=file
#                z=gzip j=bzip2 J=xz

# Standalone compression
gzip file                          # -> file.gz (removes original)
gunzip file.gz                     # decompress
bzip2 file                         # -> file.bz2
xz file                            # -> file.xz
zcat file.gz                       # view a .gz without extracting

# zip (cross-platform)
zip archive.zip file1 file2        # create a zip
zip -r archive.zip folder/         # recursive
unzip archive.zip                  # extract
unzip -l archive.zip               # list contents
```

Rule of thumb: `.tar.gz` is fast and universal; `.tar.xz` compresses smaller but slower; use `.zip` when sharing with Windows users.

### Troubleshooting

**"tar: Removing leading '/' from member names"** — this is just a warning; tar strips absolute paths so the archive extracts relative to the current directory (safer). Nothing to fix.

**Not sure which compression an archive uses** — run `file archive.ext`; it identifies gzip/bzip2/xz/zip. Modern `tar -xf archive.*` auto-detects compression, so you rarely need the `z`/`j`/`J` flag when extracting.

**Extraction dumps files all over the current directory** — the archive wasn't made with a top-level folder. Peek first with `tar -tf archive.tar | head`, then extract into a fresh dir with `-C`.

**"unexpected end of file" / corrupted archive** — the download or transfer was incomplete. Re-download and verify with a checksum (`sha256sum`). Partial recovery is sometimes possible with `gzip -d` ignoring errors, but data may be lost.

**Extraction fails with "No space left on device"** — the uncompressed data is far bigger than the archive. Check free space with `df -h` before extracting large archives.

**Permissions/owners wrong after extraction** — only root can restore original ownership. Extract as root, or accept that files become owned by you.

---