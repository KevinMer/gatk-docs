## When should I use -L to pass in a list of intervals?

By Sheila

<p>The <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_engine_CommandLineGATK.php#--intervals">-L argument</a> (short for --intervals) enables you to restrict your analysis to specific intervals instead of running over the whole genome. Using this argument can have important consequences for performance and/or results. Here, we present some guidelines for using it appropriately depending on your experimental design.</p>

<h4>In a nutshell, if you’re doing:</h4>

<p><strong>- Whole genome analysis:</strong> intervals are not required but they can help speed up analysis<br><strong>- Whole exome analysis:</strong> you must provide the list of capture targets (typically genes/exons)<br><strong>- Small targeted experiment:</strong> you must provide the targeted interval(s)<br><strong>- Troubleshooting:</strong> you can run on a specific interval to test parameters or create a data snippet</p>

<h3>Important notes:</h3>

<p>Whatever you end up using -L for, keep this in mind: for tools that output a bam or VCF file, the output file will only contain data from the intervals specified by the -L argument. To be clear, we do not recommend using -L with tools that output a bam file since doing so will omit some data from the output.</p>

<h4>Example Use of -L:</h4>

<ul><li><p><code class="code codeInline" spellcheck="false">-L 20</code> for chromosome 20 in b37/b39 build</p></li>
<li><p><code class="code codeInline" spellcheck="false">-L chr20:1-100</code> for chromosome 20 positions 1-100 in hg18/hg19 build</p></li>
<li><p><code class="code codeInline" spellcheck="false">-L intervals.list</code> (or <code class="code codeInline" spellcheck="false">intervals.interval_list</code>, or <code class="code codeInline" spellcheck="false">intervals.bed</code>) where the value passed to the argument is a text file containing intervals</p></li>
<li><p><code class="code codeInline" spellcheck="false">-L some_variant_calls.vcf</code> where the value passed to the argument is a VCF file containing variant records; their genomic coordinates will be used as intervals.</p></li>
</ul><h4>Specifying contigs with colons in their names, as occurs for new contigs in GRCh38, requires special handling for GATK versions prior to v3.6. Please use the following workaround.</h4>

<p><strong>-</strong> For example, <code class="code codeInline" spellcheck="false">HLA-A*01:01:01:01</code> is a <a rel="nofollow" href="http://gatkforums.broadinstitute.org/gatk/discussion/7857">new contig in GRCh38</a>. The colons are a new feature of contig naming for GRCh38 from prior assemblies. This has implications for using the <code class="code codeInline" spellcheck="false">-L</code> option of GATK as the option also uses the colon as a delimiter to distinguish between contig and genomic coordinates.<br><strong>-</strong> When defining coordinates of interest for a contig, e.g. positions 1-100 for chr1, we would use <code class="code codeInline" spellcheck="false">-L chr1:1-100</code>. This also works for our HLA contig, e.g. <code class="code codeInline" spellcheck="false">-L HLA-A*01:01:01:01:1-100</code>.<br><strong>-</strong> However, when passing in an entire contig, for contigs with colons in the name, you must add <code class="code codeInline" spellcheck="false">:1+</code> to the end of the chromosome name as shown below. This ensures that portions of the contig name are appropriately identified as part of the contig name and not genomic coordinates.</p>

<pre class="code codeBlock" spellcheck="false">-L HLA-A*01:01:01:01:1+
</pre>

<hr></hr><h3>So here’s a little more detail for each experimental design type.</h3>

<h4>Whole genome analysis</h4>

<p>It is not necessary to use an intervals list in whole genome analysis -- presumably you're interested in the whole genome!</p>

<p>However, from a technical perspective, you may want to mask out certain contigs (e.g. chrY or non-chromosome contigs) or regions (e.g. centromere) where you know the data is not reliable or is very messy, causing excessive slowdowns. You can do this by providing a list of "good" intervals with <code class="code codeInline" spellcheck="false">-L</code>, or you could also provide a list of "bad" intervals with <code class="code codeInline" spellcheck="false">-XL</code>, which does the exact opposite of <code class="code codeInline" spellcheck="false">-L</code>: it excludes the provided intervals. We share the whole-genome interval lists (of good intervals) that we use in our production pipelines, in our resource bundle (see Download page).</p>

<h4>Whole exome analysis</h4>

<p>By definition, exome sequencing data doesn’t cover the entire genome, so many analyses can be restricted to just the capture targets (genes or exons) to save processing time. There are even some analyses which <strong>should</strong> be restricted to the capture targets because failing to do so can lead to suboptimal results.</p>

<p>Note that we recommend adding some “padding” to the intervals in order to include the flanking regions (typically ~100 bp). No need to modify your target list; you can have the GATK engine do it for you automatically using the <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_engine_CommandLineGATK.php#--interval_padding">interval padding</a> argument. This is not required, but if you do use it, you should do it consistently at all steps where you use -L.</p>

<p>Below is a step-by-step breakdown of the Best Practices workflow, with a detailed explanation of why -L should or shouldn’t be used with each tool.</p>

<table><thead><tr><th><strong>Tool</strong></th>
  <th align="center"><strong>-L?</strong></th>
  <th><strong>Why / why not</strong></th>
</tr></thead><tbody><tr><td><strong>BaseRecalibrator</strong></td>
  <td align="center">YES</td>
  <td>This excludes off-target sequences and sequences that may be poorly mapped, which have a higher error rate. Including them could lead to a skewed model and bad recalibration.</td>
</tr><tr><td><strong>PrintReads</strong></td>
  <td align="center">NO</td>
  <td>Output is a bam file; using -L would lead to lost data.</td>
</tr><tr><td><strong>UnifiedGenotyper/Haplotype Caller</strong></td>
  <td align="center">YES</td>
  <td>We’re only interested in making calls in exome regions; the rest is a waste of time &amp; includes lots of false positives.</td>
</tr><tr><td><strong>Next steps</strong></td>
  <td align="center">NO</td>
  <td>No need since subsequent steps operate on the callset, which was restricted to the exome at the calling step.</td>
</tr></tbody></table><h4>Small targeted experiments</h4>

<p>The same guidelines as for whole exome analysis apply except you do not run BQSR on small datasets.</p>

<h4>Debugging / troubleshooting</h4>

<p>You can use -L a lot while troubleshooting! For example, you can just provide an interval at the command line, and the output file will contain the data from that interval.This is really useful when you’re trying to figure out what’s going on in a specific interval (e.g. why HaplotypeCaller is not calling your favorite indel) or what would be the effect of changing a parameter (e.g. what happens to your indel call if you increase the value of -minPruning). This is also what you’d use to generate a file snippet to send us as part of a bug report (except that never happens because GATK has no bugs, ever).</p>
