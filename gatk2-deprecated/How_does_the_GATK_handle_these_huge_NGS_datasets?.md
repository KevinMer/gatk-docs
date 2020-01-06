## How does the GATK handle these huge NGS datasets?

By Geraldine_VdAuwera

<p>Imagine a simple question like, "What's the depth of coverage at position A of the genome?"</p>

<p>First, you are given billions of reads that are aligned to the genome but not ordered in any particular way (except perhaps in the order they were emitted by the sequencer).  This simple question is then very difficult to answer efficiently, because the algorithm is forced to examine every single read in succession, since any one of them might span position A.  The algorithm must now take several hours in order to compute this value.</p>

<p>Instead, imagine the billions of reads are now sorted in reference order (that is to say, on each chromosome, the reads are stored on disk in the same order they appear on the chromosome).  Now, answering the question above is trivial, as the algorithm can jump to the desired location, examine only the reads that span the position, and return immediately after those reads (and only those reads) are inspected.  The total number of reads that need to be interrogated is only a handful, rather than several billion, and the processing time is seconds, not hours.</p>

<p>This reference-ordered sorting enables the GATK to process terabytes of data quickly and without tremendous memory overhead.  Most GATK tools run very quickly and with less than 2 gigabytes of RAM.  Without this sorting, the GATK cannot operate correctly.  Thus, it is a fundamental rule of working with the GATK, which is the reason for the Central Dogma of the GATK:</p>

<h4>All datasets (reads, alignments, quality scores, variants, dbSNP information, gene tracks, interval lists - everything) must be sorted in order of one of the <a rel="nofollow" href="http://gatkforums.broadinstitute.org/discussion/1204/what-input-files-does-the-gatk-accept">canonical references sequences</a>.</h4>
