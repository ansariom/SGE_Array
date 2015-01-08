# SGE_Parallel

#### Submitting a list of commands as an array job to SGE. Easily.

Note: This software is intended for use on the [CGRB](http://shell.cgrb.oregonstate.edu) infrastructure,
and may not work well on other SGE installations. 

### Overview

So you have a bunch of commands that you want to run via SGE, maybe in a text file called `commands.txt`:

```
runAssembly sample_117.fasta -o sample_117.fasta.out
runAssembly sample_162.fasta -o sample_162.fasta.out
runAssembly sample_169.fasta -o sample_169.fasta.out
runAssembly sample_30.fasta -o sample_30.fasta.out
runAssembly sample_34.fasta -o sample_34.fasta.out
runAssembly sample_38.fasta -o sample_38.fasta.out
runAssembly sample_47.fasta -o sample_47.fasta.out
runAssembly sample_58.fasta -o sample_58.fasta.out
runAssembly sample_96.fasta -o sample_96.fasta.out
```

Or maybe you generated your commands from the list of file names with come clever usage of awk or sed (I generated
the above with `ls -1 *.fasta | awk '{print "runAssembly " $1 " -o " $1 ".out"}' > commands.txt`). From
there you could just make the file an executable shell script, or even pipe it right into bash with a
`cat commands.txt | bash`. But what if you want to run these via SGE?

Probably, you should run them as an array job (because using a loop to submit them as individual jobs is NOT GOOD, ok?),
but this means using the clunky `$SGE_TASK_ID` syntax, which will only take numerals. `SGE_Parallel` is
to your rescue: it takes a list of commands (either as a file, or on stdin) and turns them into an array job. Boom.

```
cat commands.txt | SGE_Parallel

# or

SGE_Parallel -c commands.txt

# what about?

ls -1 *.fasta | awk '{print "runAssembly " $1 " -o " $1 ".out"}' | SGE_Parallel
```

### Reasonable Defaults and Cool Features

By default, each command is run requesting 4 gigs of RAM, and will be killed if it exceeds that. Each command
will be killed if it attempts to create a file of more than 500G. The maximum number of commands that can
run simultaneously across any number of machines is limited to 50 (to preserve network resources, this can
be changed for IO-light commands). 

The default log directory (where the qsub submit script is created, and where the stdout and stderr of
each command are written) is `jYEAR-MON-DAY_HOUR-MIN-SEC_cmd_etal`, as in `j2015-01-07_16-35-67_runAssembly_etal`,
so that you can easily organize your log information by time, and move/remove things in an orderly fashion
(you know, like to delete all of today's work, `rm -f j2015-01-07*`). These log directories are compatible
with the already existing `SGE_Plotdir` which reports RAM and time usage for each command in the array job.

Most of this can be changed, here's the help output:

```
usage: SGE_Parallel [-h] [-c COMMANDSFILE] [-q QUEUE] [-m MEMORY]
                    [-f FILELIMIT] [-b CONCURRENCY] [-P PROCESSORS]
                    [-r RUNDIR] [-p PATH] [-v] [--showchangelog]

Runs a list of commands specified on stdin as an SGE array job. Example usage:
`cat commands.txt | SGE_Parallel` or SGE_Parallel -c commands.txt

optional arguments:
  -h, --help            show this help message and exit
  -c COMMANDSFILE, --commandsfile COMMANDSFILE
                        The file to read commands from. Default: -, meaning
                        standard input.
  -q QUEUE, --queue QUEUE
                        The queue(s) to send the commands to. Default: all
                        queues you have access to.
  -m MEMORY, --memory MEMORY
                        Amount of free RAM to request for each command, and
                        the maximum that each can use without being killed.
                        Default: 4G
  -f FILELIMIT, --filelimit FILELIMIT
                        The largest file a command can create without being
                        killed. (Preserves fileservers.) Default: 500G
  -b CONCURRENCY, --concurrency CONCURRENCY
                        Maximum number of commands that can be run
                        simultaneously across any number of machines.
                        (Preserves network resources.) Default: 50
  -P PROCESSORS, --processors PROCESSORS
                        Number of processors to reserve for each command.
                        Default: 1
  -r RUNDIR, --rundir RUNDIR
                        Job name and the directory to create or OVERWRITE to
                        store log information and standard output of the
                        commands. Default: 'jYEAR-MON-DAY_HOUR-MIN-
                        SEC_<cmd>_etal' where <cmd> is the first word of the
                        first command.
  -p PATH, --path PATH  What to use as the PATH for the commands. Default:
                        whatever is output by echo $PATH.
  -v, --version         show program's version number and exit
  --showchangelog       Show the changelog for this program.
```

It can also be used to run non-array jobs (though it will run it as an array of 1 command anyway). The cool
thing is that no longer do you have to wrap the command in quotes (unless you are doing funky things like
using environment variables right on the command-line, or you want to use | or >, which you'll have to escape).

This means shell autocompletion will work!

```
echo runAssembly input.fasta -o assembly_output | SGE_Parallel
```

If your command needs funky shell stuff, you'll have to make sure `echo` can print it properly, by escaping
or using 

```
echo runAssembly input.fasta -o	assembly_output	\> log.txt | SGE_Parallel
```


### Future Directions

I'd like to have the thing read a config file called `.sge_parallel` in `$HOME` so that the defaults
for the options (`--path`, `--queue`, `--memory` etc.) can be adjusted to minimize typing in
day-to-day usage.


