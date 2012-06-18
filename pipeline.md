--
layout: default
title: The Joshua Pipeline
--
This page describes the Joshua pipeline script, which runs the complete machine translation pipeline
for Joshua, from data preparation to testing and evaluation with BLEU.

The entry point for the pipeline is the scripts `$JOSHUA/scripts/training/pipeline.pl`, which is
designed in the spirit of Moses' `train-model.pl` (formerly `train-factored-phrase-model.perl`).
The script:

   - Runs the complete pipeline from start (training corpus normalization and tokenization) to
     finish (producing a BLEU score on a test set).

   - Caches the results of intermediate steps (based on file contents, not file existence or
     timestamps), so the pipeline can be debugged or shared across similar runs with (almost) no time
     spent recomputing expensive steps.
 
   - Can jump into the pipeline at any of a set of predefined points.

This document describes how to use this pipeline.

## QUICK START

   1. See
      [the INSTALL file](https://github.com/joshua-decoder/joshua/blob/master/scripts/training/INSTALL)
      to setup the pipeline.  If Joshua is installed, this is mostly a matter of making sure that a
      few environment variables are correctly defined.

   2. Prepare your data.  The pipeline script needs to be told where to find the raw training,
      tuning, and test data.  A good convention is to place these files in a data/ subdirectory of
      your run's working directory.  The expected format (for each of training, tuning, and test) is
      a pair of files that share a common path prefix and are distinguished by their extension:

        data/
             train.[SOURCE]
             train.[TARGET]
             tune.[SOURCE]
             tune.[TARGET]
             test.[SOURCE]
             test.[TARGET]

      These files should be parallel at the sentence level (with one sentence per line), should be
      in UTF-8, and should be untokenized (tokenization occurs in the pipeline).  [TARGET] and
      [SOURCE] denote variables that should be replaced with the actual target and source language
      abbreviations (e.g., "cz" and "en").
   
   3. Run the pipeline.  The following is the minimal invocation to run the complete pipeline:

        $JOSHUA/scripts/training/pipeline.pl  \
           --corpus data/train                \
           --tune data/tune                   \
           --test data/test                   \
           --source [SOURCE]                  \
           --target [TARGET]

      The --corpus, --tune, and --test flags define file prefixes that are concatened with the
      language extensions given by --target and --source.  Note the correspondences with the files
      created in step (2) above.  The prefixes can be either absolute or relative pathnames.  This
      particular invocation assumes that a subdirectory data/ exists in the current directory, that
      you are translating from a language identified by the [SOURCE] extension to a language
      identified by the [TARGET] extension, that the training data can be found at data/train.en and
      data/train.ur, and so on).

      Assuming no problems arise, this command will run the complete pipeline, producing BLEU scores
      at the end.  The next section describes further available command-line options.

## COMPLETE LIST OF COMMAND-LINE OPTIONS

    pipeline.pl

- `--rundir DIR`

    Place all files beneath the specified directory.  The default is the current directory.

- `--first-step STEP`
- `--last-step STEP`
  
    Allows starting and stopping at at of the following steps: 

    - ALIGN (alignment of the training data)
    - PARSE (parsing of the target side for SAMT grammar extraction)
    - THRAX (grammar extraction)
    - MERT (tuning)
    - TEST (decoding a test set)

- `--source SOURCE`
- `--target TARGET`

    Specifies the source and target language extensions.  

- `--corpus CORPUS1`
- `[--corpus CORPUS2]`
- `[...]`

    Specifies a file prefix for a training corpus.  If this flag is found multiples times, the corpora are all concatenated together for alignment and grammar extraction.  The use of multiple `--corpora` flags is only supported when running the script from the beginning; if you are skipping steps, only a single corpus can be provided (so you must do the concatenation yourself, if you so need). 

- `--tune PREFIX`
- `--test PREFIX`
  
    Specify file prefixes for tuning and test data.  Unlike --corpus,
    only one instance each of --tune or --test is allowed (if you
    provide multiple ones, only the last value is used).

- `--type {hiero,samt}`

    Whether to learn a Hiero or SAMT grammar.  The default is Hiero.

- `--alignment FILE`

    Provide an alignment for the training data.  The format is the
standard format where 0-indexed many-many alignment pairs for a
sentence are provided on a line, source language first, e.g.,

    0-0 0-1 1-2 1-7 ...

    This value is required if you are skipping the alignment step.

-  `--mbr [default]`
-  `--nombr`

    Do (not) do MBR reranking of the n-best output of the test data.

-  `--lmfile FILE`

    Use the specified file as the language model for decoding (for
    both tuning and decoding of the test set).  If not provided,
    SRILM is used to build a Kneser-Ney interpolated 5-gram language
    model from the target side of the training data.

-  `--filter-lm [default]`
-  `--no-filter-lm`

    Use Kenneth Heafield's "filter" program to filter the language
    model to the training data.  This is only available if a training
    corpus was provided.

- `--maxlen LEN [default=50]`

    Remove parallel sentences from the training data that are longer
    than this length (on either side).

- `--grammar FILE`

    Use the specified grammar instead of learning one with Thrax.

- `--joshua-config TEMPLATE`

    Use the specified file as the Joshua config file instead of the
    default template.  This file is a template into which are
    substituted run-specific information; see the TEMPLATE section
    below for more information.

- `--decoder-command TEMPLATE`

    Use the specified file as the decoding command.  This file is a
    template into which are substituted run-specific information; see
    the TEMPLATE section below for more information.

- `--thrax-conf FILE`

    Use the provided Thrax configuration file instead of the (grammar-specific) default.

- `--no-subsample [default]`
- `--subsample`

    Subsampling is a means of throwing away a portion of the training
    data by only retaining sentence pairs that appear relevant to the
    tuning and development data.  This makes the pipeline run
    significantly faster (particularly alignment), and often comes at
    only a small cost in accuracy on the development set.  Subsampling
    works by comparing the training data to the tuning and test sets
    passed in with --tune and --test.

- `--joshua-mem MEM [3100m]`

    Provide the maximum heap size available to instances of the Joshua
    decoder.  Uses Java notation (the value here is passed to Java's
    -Xmx flag).

- `--hadoop-mem MEM [4g]`

    Provide the maximum heap size for the hadoop instances used to
    learn the grammar in Thrax.

- `--jobs N`

    The number of parallel Joshua decoders to start (via qsub, by default) when decoding a input file, during both tuning and decoding.

- `--qsub-args "ARGS"`

    Provide the specified qsub arguments to the Joshua decoder command.  Only used if the Joshua decoder_command file contains the appropriate template variable (see the TEMPLATES section below).

## STEPS

The Joshua pipeline provides support for two major use cases.

1. Running the pipeline from start to finish.  If problems arise somewhere in the pipeline, it can be quickly rerun due to CachePipe. 

2. Running pieces of the pipeline.  Early steps can be skipped with --first-step (e.g., if you already have an aligned corpus or grammar you wish to decode with), and –-last-step can be used to exit early from the script.  The following steps are defined:

    - *FIRST*: The beginning of the script.
    - *SUBSAMPLE*: Subsampling of the corpus towards the tuning and test data.
    - *ALIGN*: Alignment of the training data.
    - *PARSE*: Parsing of the target language data (for SAMT grammar extraction only).
    - *THRAX*: Grammar extraction.
    - *MERT*: Tuning.
    - *TEST*: Decoding a test set.

## TEMPLATES

A few pieces of the pipeline are subject to enough variation across runs and across computing environment that it is easier to provide template files than to use command-line arguments.  The Joshua pipeline allows the Joshua configuration file and the Joshua decoder command to be templatized.  Here are the options available to those files:

- `<JOSHUA>` is the root of the Joshua installation ($JOSHUA env. var.)
- `<INPUT>` is the file being decoded
- `<OUTPUT>` is the output file that the decoder command creates
- `<SOURCE>` is the source language (--source)
- `<TARGET>` is the target language (--target)
- `<LMFILE>` is the language model
- `<MEM>` is the amount of memory available to the Joshua decoder instances
- `<OOV>` is the OOV tag used in the grammar ("OOV" for SAMT, "X" for Hiero)
- `<NUMJOBS>` is the degree of parallelization for the Joshua decoder command
- `<QSUB_ARGS>` is the qsub arguments (--qsub-args)
- `<REF>` is the reference file
- `<CONFIG>` is the location of the joshua configuration file
- `<LOG>` is where the Joshua decoder should put its log file

The default configuration file is the following, located in `$JOSHUA/scripts/training/templates/mert/decoder_command.qsub`.  It can be overridden with the --decoder_command flag.

    cat <INPUT> | awk 'BEGIN { num = 0 } {print "<seg id=\"" num "\">" $0 "</seg>"; \
    num++}' | <JOSHUA>/scripts/training/parallelize/parallelize.pl -j <NUMJOBS> -m \
    --qsub-args '<QSUB_ARGS>' -- java -Xmx<MEM> -Djava.library.path=<JOSHUA>/lib \
    -Dfile.encoding=utf8 -cp <JOSHUA>/bin/:<JOSHUA>/lib/thrax.jar \
    joshua.decoder.JoshuaDecoder <CONFIG> > <OUTPUT> 2> <LOG>

In addition, if --jobs is set to 1, the following template (located at `$JOSHUA/scripts/training/templates/mert/decoder_command.sequential`) is selected instead:

    cat <INPUT> | java -Xmx<MEM> -Djava.library.path=<JOSHUA>/lib \
    -Dfile.encoding=utf8 -cp <JOSHUA>/bin/:<JOSHUA>/lib/thrax.jar joshua.decoder.JoshuaDecoder \
    <CONFIG> > <OUTPUT> 2> <LOG>

## COMMON USE CASES AND PITFALLS 

- Memory usage is a major consideration in decoding with Joshua and hierarchical grammars.  In particular, SAMT grammars often require a large amount of memory.  Many steps have been taken to reduce memory usage, including beam settings and test-set- and sentence-level filtering of grammars.  However, memory usage can still be in the tens of gigabytes.

    To accommodate this kind of variation, the pipeline script allows you to specify both (a) the amount of memory used by the Joshua decoder instance and (b) the amount of memory required of nodes obtained by the qsub command.  These are accomplished with the --joshua-mem MEM and --qsub-args ARGS commands.  For example,

    pipeline.pl --joshua-mem 32g --qsub-args "-l pvmem=32g -q himem.q" ...

## FEEDBACK 

Please email joshua_technical@googlegroups.com with problems or
suggestions.  
