## I expect to see a variant at a specific site, but it's not getting called

By Geraldine_VdAuwera

<p>This can happen when you expect a call to be made based on the output of other variant calling tools, or based on examination of the data in a genome browser like IGV.</p>

<p>There are several possibilities, and among them, it is possible that GATK may be missing a real variant. But we are generally very confident in the calculations made by our tools, and in our experience, most of the time, the problem lies elsewhere. So, before you post this issue in our support forum, please follow these troubleshooting guidelines, which hopefully will help you figure out what's going on.</p>

<p>In all cases, to diagnose what is happening, you will need to look directly at the sequencing data at the position in question.</p>

<h3>1. Generate the bamout and compare it to the input bam</h3>

<p>If you are using HaplotypeCaller to call your variants (as you nearly always should) you'll need to run an extra step first to produce a file called the "bamout file". See <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/article?id=5484">this tutorial</a> for step-by-step instructions on how to do this.</p>

<p>What often happens is that when you look at the reads in the original bam file, it looks like a variant should be called. However, once HaplotypeCaller has performed the realignment, the reads may no longer support the expected variant. Generating the bamout file and comparing it to the original bam will allow you to elucidate such cases.</p>

<p>In the example below, you see the original bam file on the top, and on the bottom is the bam file after reassembly. In this case, there seem to be many SNPs present, however, after reassembly, we find there is really a large deletion!</p>

<p><a rel="nofollow" href="https://us.v-cdn.net/5019796/uploads/FileUpload/cf/d2aa18df0a32463bfae7ef5eda101c.png"><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/cf/d2aa18df0a32463bfae7ef5eda101c.png" alt="image" class="embedImage-img importedEmbed-img"></img></a></p>

<h3>2. Check the base qualities of the non-reference bases</h3>

<p>The variant callers apply a minimum base quality threshold, under which bases will not be counted as supporting evidence for a variant. This is because low base qualities mean that the sequencing machine was not confident that it called the right bases. If your expected variant is only supported by low-confidence bases, it is probably a false positive.</p>

<p>Keep in mind that the depth reported in the DP field of the VCF is the unfiltered depth. You may believe you have good coverage at your site of interest, but since the variant callers ignore bases that fail the quality filters, the actual coverage seen by the variant callers may be lower than you think.</p>

<h3>3. Check the mapping qualities of the reads that support the non-reference allele(s)</h3>

<p>The quality of a base is capped by the mapping quality of the read that it is on. This is because low mapping qualities mean that the aligner had little confidence that the read was mapped to the correct location in the genome. You may be seeing mismatches because the read doesn't belong there -- in fact, you may be looking at the sequence of some other locus in the genome!</p>

<p>Keep in mind also that reads with mapping quality 255 ("unknown") are ignored.</p>

<h3>4. Check how many alternate alleles are present</h3>

<p>By default the variant callers will only consider a certain number of alternate alleles. This parameter can be relaxed using the <code class="code codeInline" spellcheck="false">--max_alternate_alleles</code> argument  (see <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_haplotypecaller_HaplotypeCaller.php">the HaplotypeCaller documentation page</a> to find out what is the default value for this argument). Note however that genotyping sites with many alternate alleles increases the computational cost of the processing, scaling exponentially with the number of alternate alleles, which means it will use more resources and take longer. Unless you have a really good reason to change the default value, we highly recommend that you not modify this parameter.</p>

<h3>5. When using UnifiedGenotyper, check for overlapping deletions</h3>

<p>The UnifiedGenotyper ignores sites if there are too many overlapping deletions. This parameter can be relaxed using the <code class="code codeInline" spellcheck="false">--max_deletion_fraction</code> argument (see <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_genotyper_UnifiedGenotyper.php">the UG's documentation page</a> to find out what is the default value for this argument) but be aware that increasing its value could adversely affect the reliability of your results.</p>

<h3>6. Check for systematic biases introduced by your sequencing technology</h3>

<p>Some sequencing technologies introduce particular sources of bias. For example, <br>
in data produced by the SOLiD platform, alignments tend to have reference bias and it can be severe in some cases. If the SOLiD reads have a lot of mismatches (no-calls count as mismatches) around the the site, you are probably seeing false positives.</p>

<h3>7. Try fiddling with graph arguments (ADVANCED)</h3>

<p>This is highly experimental, but if all else fails, worth a shot (with HaplotypeCaller and MuTect2).</p>

<h4>Fiddle with kmers</h4>

<p>In some difficult sequence contexts (e.g. repeat regions), when some default-sized kmers are non-unique, cycles get generated in the graph. By default the program increases the kmer size automatically to try again, but after several attempts it will eventually quit trying and fail to call the expected variant (typically because the variant gets pruned out of the read-threading assembly graph, and is therefore never assembled into a candidate haplotype). We've seen cases where it's still possible to force a resolution using <code class="code codeInline" spellcheck="false">-allowNonUniqueKmersInRef</code> and/or increasing the <code class="code codeInline" spellcheck="false">--kmerSize</code> (or range of permitted sizes: 10, 25, 35 for example).</p>

<h5>Note: While --allowNonUniqueKmersInRef allows missed calls to be made in repeat regions, it should not be used in all regions as it may increase false positives. We have plans to improve variant calling in repeat regions, but for now please try this flag if you notice calls being missed in repeat regions.</h5>

<h4>Fiddle with pruning</h4>

<p>Decreasing the value of <code class="code codeInline" spellcheck="false">-minPruning</code> and/or <code class="code codeInline" spellcheck="false">-minDanglingBranchLength</code> (i.e. increasing the amount of evidence necessary to keep a path in the graph) can recover variants, at the risk of taking on more false positives.</p>
