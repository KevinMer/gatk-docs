## At what point should I merge read group BAM files belonging to the same sample into a single file?

By Sheila

<p>It is fairly common to have multiple read groups for a sample, either from sequencing multiple libraries or from spreading a library across multiple lanes. It seems this causes a lot of confusion, and people often tell us they're not sure how to organize the data for the pre-processing steps or how to feed the data into HaplotypeCaller.</p>

<p>Well, there are several options for organizing the processing. We have a fairly detailed FAQ article that describes <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/article?id=3060">our preferred workflow for pre-processing data from multiplexed sequencing and multi-library designs</a>. But in this article we describe at a simpler level what are the main two options depending on how you want to provide the analysis ready BAM files to the variant caller.</p>

<h3>To produce a combined per-sample bam file to feed to HaplotypeCaller (most common)</h3>

<p>The simplest thing to do is to input all the bam files that belong to that sample, either at the MarkDuplicates step, the Indel Realignment step or at the BQSR step. The choice depends mostly on how deep the coverage is. High depth means a lot of data to process at the same time, which slows down Indel Realignment. This is because Indel Realignment ignores all read group information and simply processes all reads together. BQSR doesn't suffer from that problem because it processes read groups separately. In either case, when you input all samples together, the bam that gets written out with the processed data will include all the libraries / read groups in one handy per-sample file.</p>

<p><em>Note: We do not require the PU field in the RG, however, BQSR will consider the PU field over all other fields.</em></p>

<h3>To produce a separate bam file for each read group (less common)</h3>

<p>Another option is to keep all the bam files separate until variant calling, and then input them to Haplotype Caller together. You can do this by simply running Indel Realignment and BQSR on each of the bams separately. You can then input all of the bams into HaplotypeCaller at once. This works even if you want to run HaplotypeCaller in GVCF mode, which can only be done on a single sample at a time. As long as the SM tags are identical, HaplotypeCaller will recognize that it's a single-sample run. This is because the GATK engine will merge the data before presenting it to the HaplotypeCaller tool, so HaplotypeCaller does not know nor care whether the data came from many files or one file.</p>

<p><em>Note: If you input many bam files into Indel Realigner, the default output is one bam file. However, you can output one bam file for each input bam file by using <a rel="nofollow" href="https://www.broadinstitute.org/gatk/gatkdocs/org_broadinstitute_gatk_tools_walkers_indels_IndelRealigner.php#--nWayOut"><code class="code codeInline" spellcheck="false">-nWayOut</code></a>.</em></p>
