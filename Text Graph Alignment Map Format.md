#Text Graph Alignment Map (TGAM) Format Specification

This document defines a file format, the **Text Graph Alignment Map** or **TGAM** format, to be used by participants in the GA4GH HGVM Graph Aligner Bake-Off. In the bake-off, participants will create Docker containers for their aligners, which take as input a GA4GH graph server URL and a paired-end FASTQ file, and which produce output in TGAM format. A TGAM file defines **alignments** between a collection of **reads**, grouped into **fragments** (also known as **templates**, generally with two reads each), and a **side graph**. Each read may have zero or more alignments, and all alignments are specified in side graph coordinates.

Here is an example of TGAM format.
```
# NAME	IS_SECONDARY	IS_REVERSE	SCORE	MAPQ	PATH	SEQ	QUAL	PREV_NAME	NEXT_NAME	SAMPLE_NAME	READ_GROUP
frag1read1	false	false	100	20	1:0:false:5/5|1/1/A|4/4,2:2:true:3/3	GATTAAATATCCC	QQQQQQQQQQQQQ	*	frag1read2	Franklin	FranklinRG1
frag1read2	false	true	75	20	2:20:false:3/3|0/1/A|3/3|1/0|4/4	CTCATAGGAG	QQQQQQQQQQ	frag1read1	*	Franklin	FranklinRG1
```

##Overview

TGAM is a line-oriented, tab-separated format, in which, as in SAM, `*` is used to represent empty field values. Lines beginning with `#` are comments. Lines beginning with `@` are interpreted as header lines, and are allowed, but their behavior is implementation-defined.

Each line in a TGAM file represents an alignment. (Note that some alignments--those with no `PATH`--represent a lack of alignment to anything in the reference, and should be produced for a read if and only if no other alignments are produced. Each alignment is associated with a particular read, and that read is associated with a particular fragment.

##Field definitions

The fields are defined as follows:

* **NAME**: Name of the read. Unlike in SAM, where fragments are named, in TGAM each individual read has its own name, which uniquely identifies the read being aligned. This might be the same as the read's FASTQ header ID. May not be `*` or begin with `#`.

* **IS_SECONDARY**: Should be `true` if the alignment is a secondary alignment, and `false` otherwise. Every read in a file should have exactly one primary alignment, which is the best of all the reads' alignments, with the rest being secondary.

* **IS_REVERSE**: Should be `true` if the sequence output in `SEQ` is the reverse complement of the original input sequence, and `false` otherwise. Generally determined by whether the required orientation of the `PATH` agrees with the sequencing orientation.

* **SCORE**: Aligner-defined integer score for how good the alignment is. Higher is better. Will be `*` if the read is not mapped, or if scores are not calculated by the aligner.

* **MAPQ**: Mapping quality score as a decimal value. Will be `*` if the read is not mapped, or if mapping qualities are not calculated by the aligner. It is -10 * log10(probability that the `PATH` provided is not the path that produced this read). It should account correctly for the aligner's error model (i.e., the case that, instead of being generated by the given path, the read was generated from a different path and was read with one or more errors) and for multimapping (i.e. the case that, instead of being generated by the given path, the read was generated by a different path that spells out the same sequence).

* **PATH**: Path in the graph to which the read is aligned. Will be `*` if and only if the read is not mapped. The path takes the form of a comma-separated list of colon-separated **mapping** 4-tuples, each of which specifies an alignment between a contiguous run in the read and a sequence in the graph reference. Each mapping 4-tuple contains:
    * The ID of a sequence in the graph reference.
    * The 0-based leftmost base along that sequence at which the alignment begins.
    * A reverse flag, which is `false` if the alignment starts at that base and proceeds along the sequence's forward strand, and `true` if the alignment starts at that base and proceeds along the sequence's reverse strand.
    * A series of **edits**, separated by `|` characters.
    
    Each edit roughly plays the role of a CIGAR operation. An edit relates a possibly empty subsequence of the read to a possibly empty subsequence of the reference. It consists of a number of bases in the reference, a number of bases in the read, and, possibly, a piece of sequence giving the read's sequence for the edit, all of which are separated by `/` characters. There are four types of edits:
    * **matches**: reference base count = read base count, no read sequence given. For example: `5/5`.
    * **mismatches**: reference base count = read base count, read sequence given that does not match the corresponding reference sequence. For example: For example: `2/2/CA`.
    * **deletions**: reference base count > 0, read base count = 0, no read sequence given. For example: `3/0`.
    * **insertions**: reference base count = 0, read base count > 0, read sequence given containing inserted bases. For example: `0/3/AAA`.

    All edits must be described as one of the above four types; other types of edits are illegal. The total read base count of all edits in all mappings must be equal to the length of `SEQ` and `QUAL`.

* **SEQ**: The sequence of the alignment's read, in the orientation specified by `FLAGS`, which is also the correct orientation for interpreting `CIGAR` against the path in `PATH`. Must be expressed in upper case. Must not be `*`.

* **QUAL**: The quality scores for the bases given in `SEQ`, in the same orientation as `SEQ`, expressed in FASTQ-alike phred+33 format. May be `*` if qualities are unavailable.

* **PREV_NAME**: The name of the previous read on this read's fragment, or `*` if there is no such read. The fragment is taken as being oriented 5' to 3' for the purpose of determining "previous".

* **NEXT_NAME**: The name of the next read on this read's fragment, or `*` if there is no such read. The fragment is taken as being oriented 5' to 3' for the purpose of determining "next".

* **SAMPLE_NAME**: The name of the sample from which the read was obtained.

* **READ_GROUP**: The name of the read group that the read belongs to.

##Notable Differences from SAM

TGAM format is partially based on SAM format, with several important changes:

1. Mapping is now to a side graph, instead of a linear reference. Mapping positions have been replaced by paths in the side graph.
2. Each read has a name, rather than having names be assigned to fragments.
3. Split alignments are now to be expressed on a single line, through the use of the `PATH` field.
4. CIGAR string information is now contained in the `PATH`.
5. `PREV_NAME` and `NEXT_NAME` link fragments; fragments are not considered circular.
6. No headers or tags are defined.
