## Archival

Category: Reversing

### Problem
Check out this weird file I found. I think this program has something to do with it too?

### Initial Thoughts
This problem comes with three files: `extract`, `extract.c` and `arc.bin`. Upon initial examination, the purpose of each is straightforward - `extract` is used to pull data out of `arc.bin` and `extract.c` is just the uncomplied version of `extract`. So let's try doing just that:

```
~ ./extract arc.bin out-directory
~ ls -l out-directory
total 884
-rw-rw-r-- 1 lauren lauren    629 Aug  9 16:11 aaaa.png
-rw-rw-r-- 1 lauren lauren 457093 Aug  9 16:11 paint.png
-rw-rw-r-- 1 lauren lauren   3718 Aug  9 16:11 sentence.png
-rw-rw-r-- 1 lauren lauren 435718 Aug  9 16:11 yeehaw.png
```

We get four perfectly normal files out. No flags yet.


### Digging Deeper
The first thought is that there might be hidden data inside `arc.bin` that wasn't extracted by `extract`.

```
~ ls -l .
total 904
-rw-rw-r-- 1 lauren lauren 899967 Aug  9 15:43 arc.bin
-rwxrwxr-x 1 lauren lauren  12976 Aug  9 15:41 extract
-rw-rw-r-- 1 lauren lauren   2602 Aug  9 15:43 extract.c
drwxrwxr-x 2 lauren lauren   4096 Aug  9 16:11 out-directory
~ ls -l out-directory
total 884
-rw-rw-r-- 1 lauren lauren    629 Aug  9 16:11 aaaa.png
-rw-rw-r-- 1 lauren lauren 457093 Aug  9 16:11 paint.png
-rw-rw-r-- 1 lauren lauren   3718 Aug  9 16:11 sentence.png
-rw-rw-r-- 1 lauren lauren 435718 Aug  9 16:11 yeehaw.png
```

From the above, we can see from above that the size of `arc.bin` is 899967. The total size of the four outputted files is 629 + 457093 + 3718 + 435718 = 897158. That leaves enough space for an extra file even. Let's run `strings` to see if we can glean any information about where this hidden data might be.  

```
~ strings arc.bin
yeehaw.png
v.)RWM=
?yu-
Yq3P
P~~j
dP>"
...
```

There's too much junk to scroll through quickly without running the risk of missing something. However, we can see the name of the first file quite clearly, so it stands to reason that we might be able to see the names of any hidden files.

```
~ strings arc.bin | grep ".png"
yeehaw.png
aaaa.png
sentence.png
paint.png
flag.png
```

Aha! We found `flag.png`. Now how do we extract it?

```
int main(int argc, char **argv) {
    
    ...

    long sz = 0;
    char *buff = read_file(argv[1], &sz);

    ...

    int filecnt = *(int *) buff;
    int *fileoffs = ((int *) buff) + 1;

    ...

    for (int i = 0; i < filecnt; i++) {
        fblk_t blk;
        if (parse_fblk(buff, sz, fileoffs[i], &blk)) {
            puts("Error parsing file");
            free(buff);
            return 1;
        }

        FILE *fp = fopen(blk.name, "wb");

        ...

        if (fwrite(blk.data, 1, blk.length, fp) != (size_t) blk.length) {
            puts("Error writing output file");
            free(buff);
            return 1;
        }

        fclose(fp);
    }

    free(buff);
}
```

Looking at the `main` function of `extract.c` above (with some lines omitted for brevity), we see that the number of files and the offsets used to find the files are defined in the header of `arc.bin`.

`
00000000   04 00 00 00  6A B7 06 00  3A A6 06 00  CF A8 06 00  20 00 00 00  62 53 68 F4  D6 40 25 9C  ....j...:....... ...bSh..@%.
0000001C   E6 BC 05 82  17 A6 06 00  E1 91 79 65  65 68 61 77  2E 70 6E 67  00 C1 68 D6  AF 9B EC 9B  ..........yeehaw.png..h.....
00000038   FB 91 E1 9C  E1 D9 A8 C3  A5 91 E1 B1  E2 91 E1 4D  E3 93 E9 91  E1 01 E1 D2  D4 91 08 91  ...............M............
00000054   E1 E2 E0 D6  B3 91 A3 5F  4F 78 FD 91  E1 91 C1 D5  A8 C5 A0 CF  99 2C 6D 0B  8C A9 0A A2  ......._Ox...........,m.....
00000070   4D B9 E9 E6  08 5E 1E F1  4D A0 B7 2A  6A 87 BC 96  07 DB E1 E2  4F 5F 1B 3B  49 18 B3 C0  M....^..M..*j.......O_.;I...
0000008C   CC 91 1D DB  C8 8C 77 AE  1F 66 1E B4  1E 7E 88 18  CC 59 E1 87  54 BF A8 B1  61 D1 E1 3F  ......w..f...~...Y..T...a..?
000000A8   F3 13 A3 D8  C5 9A 61 CD  BD 35 F5 D8  CC B5 F3 EC  71 01 6E AA  89 0C 59 BD  88 24 4D CE  ......a..5......q.n...Y..$M.

...
```

Opening up `arc.bin` in `hexedit`, we see the number of files (`04 00 00 00`), followed by the offsets for each (`6A B7 06 00`, `3A A6 06 00`, `CF A8 06 00`, `20 00 00 00`). We don't need all five files, so let's just change the offset for the last file (`6A B7 06 00`) to be at the start of where we found `flag.png`.

The ASCII for `flag.png` starts at `0F B1 0D 00`. However, when looking at the offset for the first file, `yeehaw.png`, we see that the ASCII actully appears 6 bytes after the offset indicates. That is, "yeehaw.png" is at `26 00 00 00`, but the offset for that file indicates `20 00 00 00`. So we need to adjust the offset for `flag.png` to be `09 B1 0D 00`.

Replacing `6A B7 06 00` with `09 B1 0D 00` and the re-running `extract` gives us a file called `flag.txt`, which, when we use `strings` on it, spits out "l05t_buT_n0t_f0rGotT3n_18a9b735". Wrap that in uscg{...} and we've got our flag!
