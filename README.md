# NIST Statistical Test Suite

## TL;DR

* **Step 0**: Generate raw binary data into a file, called say: randdata

```
    By "raw binary data" we mean that if your generator had
    generated these binary bits:

        1 0 0 1 0 1 1 0

    you would write/output a single byte of data with the value:

        0x96
```

* **Step 1**: Compile STS the code

```sh
    make clobber all

    BTW: Feel free to install sts if you wish:

    make install
```

* **Step 2**: Run the code

```sh
    sts randdata
```


* **Step 3**: Read the results:

```sh
    cat result.txt
```

* **Step 4**: Read about any test that may have failed in the PDF file:

```
    docs/SP800-22rev1a-improved.pdf
```

## UPDATES

**NOTE: Recent changes to this README.md file on 2025 Jan 20**:

* The [Google Drive sts-data folder][generatordata] link has been changed to allow public access.
* Added a "_p.s._" about LavaRnd at the bottom.
* Added link to the [improved SP800-22Rev1a paper][improved-paper].
* Added a TL;DR section at the top.

This project is a considerably improved version of the [NIST Statistical Test Suite][site] (**STS**), a collection of tests used in the evaluation of the randomness of bitstreams of data.

## Purpose

STS can be useful in:

- Evaluating the randomness of bitstreams produced by hardware and software key generators for cryptographic applications.
- Evaluating the quality of pseudo random number generators used in simulation and modeling applications.

## Usage

### Requirements

STS version 3 requires the external library [fftw3][fftw] to be installed in your system.
This library is also available to install in most of the package managers with the name _fftw3_.
We recommend that you compile STS version 3 with version 3.3.3 or later of fftw.

If you are not able to install fftw3 in your system, but you still want to use STS, you can compile
the program with the command `make legacy` instead of `make`. This command will make STS use another
algorithm to compute the discrete fourier transform, which is slower but does not require external libraries.

### Get data to test

As mentioned above, STS has been developed with the goal of testing the randomness of data. Therefore, if you want to use STS,
the first thing to do it to get some data generated by a pseudo-random number generator.

The [Google Drive sts-data folder][generatordata] contains some data generated by the 9 built-in PRNGs originally provided by
NIST as part of STS, for everyone to test (refer to the README.txt there for more information on it).

### How to run

For a basic usage of STS, there is only one parameter you have to choose, which is the _number of iterations_.

The data will be processed in _number of iterations_ chunks of a predefined _length of a single bitstream_ (which, by default,
is 2^20 = 1048576).

Therefore, it's required that the size of the data you use as input is at least:

    [(number of iterations) * (length of a single bitstream) / 8] BYTES

When the _number of iterations_ is high, the the results will be more accurate but the the test will be slower.

Once you have chosen the data to test and the number of iterations and cloned the repository, `cd` into the sts folder, run `make`
and then you can use the program as follows:

```sh
$ ./sts -v 1 -i 32 -I 1 -w . -F r /path/to/random/data
```

Here is the meaning of the flags and the argument used in the example command:

```
[-v 1] indicates that we want a verbosity level of 1 for the printed output (optional)
[-i 32] indicates that we want to run 32 iterations (choose bigger numbers when possible)
[-I 1] indicates that we want the program to tell us every time it finishes testing 1 bitstream (optional)
[-w .] indicates the path of the folder where you want to store the testing results
[-F r] indicates that that data will be read as raw binary data (a --> files of ASCII '0'/'1' characters)
[/path/to/random/data] indicates the path of the data we want to use as input
```

By default, STS will use as many threads as the number of cores of the machine where it runs (to speed up the processing).
If you want to specify a custom number of threads to use, you can do that with the `-T numOfThreads` additional flag.
If you want to disable multi-threading, use the `-T 1` flag.

After the run is completed a report will be generated in a file called `result.txt`.

__NB__: When `make legacy` is used, the compiled program to execute will be called `sts_legacy_fft` instead of `sts`.

__NB__: When a data file of `-` (single dash) is used, test data is read from standard input (stdin).
Job number (`-j jobnum`) based seeking into the data is disabled when test data is read from standard input.

__NB__: For more information on the usage run `./sts -h`

### [Advanced] How to run in distributed mode

If you are willing to test a huge dataset (say 1TB  of input data), which could take a long time on a single computer,
you might want to try the new STS distributed modes of operation.

The distributed mode allows the user to run sts to test a specific part of the input data and save the resulting p-values of the
testing to an output file.

Thus the user can test different chunks of the input data with different machines and later collect all the
p-values and assess them with one final run of sts in ASSESS_ONLY mode.

Here is a tutorial for the distributed more:

1. Decide how many bitstreams of the input data you want to test in each host.
2. Assign to each host a number from `0` to `k - 1` (where k is the number of hosts that you want to use).
3. Run sts with the chosen number of bitstreams in each of the hosts, using the `-m i` mode of operation (which tells STS to
perform the testing and save the p-values in an output file, without assessing them), and with the `-j X` flag, where `X` indicates
the number of the node that you chose in point 2.
4. Collect the `.pvalues` file generated by each node and save them in a folder.
5. Run sts again with the using the `-m a` mode of operation (which tells STS to assess the results of previous tests), with the
`-d pvaluesfolder` flag (where pvaluesfolder is the folder containing the binary files collected at point 4).

This way you can literally split the testing of some big input data among multiple hosts.

#### Example:

Consider the case of 32 hosts called node00, node01, ... node30, and node31 respectively.
Assume each host has a copy of the test data under /random/data or that /random is an NFS mount and all hosts have
access to the same data file.

Assume that /random/data contains 3200 GB of test data, where 1 GB is 1073741824 bytes.
Then each of the 32 hosts must process 100 GB of test data.

Assume we use the default length of a single bit stream of 1048576 bits which is 131072 bytes.
Then ***step 1*** tells us that each host must process 100 GB / 131072 bytes of test data or 819200 bitstreams.

Assume each host is assigned a node number corresponding to the hostname.
Then ***step 2*** tells us that node00 has a host number of 0, node01 has a host number of 1, and so on.

Assume that each host has a workDir named /random/work.
Further assume that at the end of the sts run, the contents of each host's /random/work is copied into a single directory,
or that /random is a NFS mount so that /random/work will be a directory that contains the work from each host.
Either way, at the end of the sts run, there will be 32 files containing pvalues:

    /random/work/sts.00.819200.1048576.pvalue
    /random/work/sts.01.819200.1048576.pvalue
    ..
    /random/work/sts.31.819200.1048576.pvalue

Assume we want to report progress after every 100 iterations.

Then step 3 says on each host we need execute:

```sh
$ ./sts -m i -w /random/work -j __host_number__ -i 819200 -I 100 -v 1 /random/data > /random/work/run.__host_number__ 2>&1
```

For example, if we have ssh access to each host, then we can execute:

```sh
$ for host in $(seq -w 00 31); do
ssh node$host nohup /some/path/sts -m i -w /random/work -j $host -i 819200 -I 100 -v 1 /random/data > /random/work/run.$host 2>&1 < /dev/null &
done
```

Assume we wait until all 32 hosts have completed their ***step 3*** runs.
Again, we assumed that for ***step 4***, either /random is a common NFS mounted filesystem, or that we can copy the contents of
/random/data from each host into a common /random/data filesystem.

Then for our final ***step 5***, on the host that either has NFS access to /random,
or that the contents of /random/data was copied into /random/data on the current host,
we need to execute:

```sh
$ ./sts -m a -d /random/work -w /random/work -v 1 /random/data
```

__NB__: In the above command, the final /random/data need not contain the contents of the test data.
This argument simply is used to document in the output, the common filename used across all hosts.

__NB__: The distributed mode of operation does not support generating the stats.txt and results.txt files with `-s`.

__NB__: Instead of each host reading from /random/data, sts may read from standard input (stdin)
by specifying `-` as a data file.  Because job number seeking is disabled when reading data from standard input,
a different part of the test data must be fed into each invocation of sts.

## Project structure

The STS version 3 comes with three folders:

- *src*: contains the source code of the suite
- *docs*: contains the papers explaining the underlying mathematical theory of the suite
- *tools*: contains some tools we used during the improvement of the suite and the legacy generators' code

## Improvements

Our major improvement starts from source code of the [original source code of version 2.1.2][site] and consists in several
 bug fixes, code improvements and added documentation. Here are some of our improvements of the 3rd version:

- Added an option to run tests in __batch mode__ (executed without human intervention)
- Added __multi-threading__, to speed up the processing when multiple cores are available
- Added __distributed run__ modes, which allow to run STS in multiple hosts and later collect assess the results.
- Improved the __performance__ (execution time with a single thread is now about 50% shorter)
- Fixed the implementation of the tests where bugs or inconsistencies with the paper were found
- Eliminated the use of the global variables in the tests (now a struct state is instead passed around as pointer)
- Eliminated the use of files to pass data during the execution of the suite (now memory is used instead)
- Introduced a "driver-like" function interface to test functions (instead of one monolithic function for each test)
- Added minimum __test conditions checks__ (now a test is disabled when its minimum requirements are not satisfied)
- Moved the generators from the sts code into a separate tool
- Improved the memory allocation patterns of each test
- Eliminated the use of "__magic numbers__" (numeric values that were used without explanation)
- Minimized the use of fixed size arrays, to prevent errors
- Added __checks for errors__ on return from system functions
- Added debugging, notice, warning and fatal error messages
- Fixed the use of uninitialized values (some issues were caught with valgrind)
- Improved the __precision__ of test constants (by using tools such as Mathematica and Calc)
- Improved the variable names when they were confusing or had a name different than the one in the paper
- Improved command line usage by adding __new flags__ (including a -h for usage help)
- Improved the coding style to be more consistent and easier to understand
- Added extensive and detailed __source code documentation__
- Improved the __support for 64-bit processors__
- Improved the Makefile to use best practices and to be more portable
- Improved the __program output__ and the contents of the file with the final results
- Added some comments and fixes to the NIST's paper: The [improved SP800-22 Rev 1a][improved-paper] is available.
- Improved readability of the source code
- Added the ability to read test data from standard input
- Fixed the warnings reported by compilers and lint
- Fixed memory leaks

## Features to add in the future

- Graphical visualization of the final p-values in a gnu-plot
- New (non-approximate) entropy test

## Legacy generators usage

In the original NIST code up through 3.1.2 of sts, 9 generators were built into the code.
These generators were moved into a standalone tool called generators that is located in the tools directory.

The first numeric argument to generators tool is the generator number:

1. Linear Congruential
2. Quadratic Congruential I
3. Quadratic Congruential II
4. Cubic Congruential
5. XOR based generator
6. Modular Exponentiation
7. Blum-Blum-Shub
8. Micali-Schnorr
9. SHA-1 based generator

The second numeric argument to generators tool is the number of 1 megabit (1048576 bits) chunks to write to output.

To build the generators tool:

```sh
$ cd tools
$ make generators
```

For more details, see the command line usage:

```sh
$ ./generators -h
```

For example, to test the first gigabit of the Quadratic Congruential II generator:

```sh
$ ./generators -i 64 3 1024 > /tmp/qc2.u8

$ cd ../src
$ make

$ ./sts -v 1 -i 1024 -I 32 /tmp/qc2.u8
```

Refer to the `genall` script in the tools directory for information on how we generated the data in the [Google Drive sts-data
folder][generatordata].

## Contributors

The people who have so far contributed to this major improvement of the NIST STS are:

- Landon Curt Noll ([lcn2](https://github.com/lcn2))
- Riccardo Paccagnella ([ricpacca](https://github.com/ricpacca))
- Tom Gilgan ([tgilgs](https://github.com/tgilgs))

By no means we believe we have fixed every last bug in this code. Moreover we do not doubt that we have introduced
new bugs during our transformation. For any such new bugs, we apologize and welcome contributions that fix either old bugs
or our newly introduced ones. Improvements to the documentation are also a very appreciated contribution.

Please feel invited to contribute by creating a pull request to submit the code you would like to be included.

You are very welcome to give us bug fixes and improvements in the form of a [GitHub Pull Request](https://github.com/arcetri/sts/pulls).

## License

The original software was developed at the National Institute of Standards and Technology by employees of the Federal Government
in the course of their official duties. Pursuant to title 17 Section 105 of the United States Code this software is not subject
to copyright protection and is in the public domain. Furthermore, Cisco's contribution is also placed in the public domain.
The NIST Statistical Test Suite is an experimental system. Neither NIST nor Cisco assume any responsibility whatsoever for
its use by other parties, and makes no guarantees, expressed or implied, about its quality, reliability, or any other
characteristic. We would appreciate acknowledgment if the software is used.

## Special thanks

Special thanks go to the original code developers. Without their efforts this modified code would not have been possible.
The original NIST document, [original SP800-22Rev1a paper][original-paper], was extremely valuable to us.  Nevertheless we highly recommend using our [improved SP800-22Rev1a paper][improved-paper]!

The above partial list of issues is presented to help explain why we extensively modified their original code.
Our bug fixes are an expression of gratitude for their efforts.

We also thank the original authors for making their code freely available. We saw the value of their efforts
and set about the tasks of extending their code to situations they did not intend.

## LavaRnd p.s.

For an interest application of STS, see the [Detailed Description of Test Results and Conclusions from LavaRnd](https://lavarand.com/what/nist-test.html).  It may be useful to note the _Ranking methodology_ and in particular definition of _proportional failures_ vs. _uniformity failures_.

[site]: <http://csrc.nist.gov/groups/ST/toolkit/rng/documentation_software.html>
[original-paper]: <http://csrc.nist.gov/groups/ST/toolkit/rng/documents/SP800-22rev1a.pdf> (see the improved-paper link as well)
[improved-paper]: <https://github.com/arcetri/sts/blob/master/docs/SP800-22rev1a-improved.pdf>
[fftw]: <http://www.fftw.org>
[generatordata]: https://drive.google.com/drive/folders/0B-W1rjDbzOiLSVNJWFpkeUE0b1k?resourcekey=0-ogAKUlLH_EvkGEqA461tnQ&usp=sharing
