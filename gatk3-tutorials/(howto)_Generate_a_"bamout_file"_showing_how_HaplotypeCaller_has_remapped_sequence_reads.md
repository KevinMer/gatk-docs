## (howto) Generate a "bamout file" showing how HaplotypeCaller has remapped sequence reads

By Geraldine_VdAuwera

<h4>NOTE: Although this document refers to HaplotypeCaller, you can generate the bamout from MuTect2 as well because the tool performs a similar reassembly to HaplotypeCaller.</h4>

<h3>1. Overview</h3>

<p>As you may know, HaplotypeCaller performs a local reassembly and realignment of the reads in the region surrounding potential variant sites (see the <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=4148">HaplotypeCaller method docs</a> for more details on why and how this is done). So it often happens that during the calling process, the reads get moved to different mapping positions than what you can observe in the BAM file that you originally provided to HC as input.</p>

<p>These remappings usually explain most discordances between calls that are expected based on the original data and actual calls made by HaplotypeCaller, so it's very useful to be able to visualize what rearrangements the tool has made.</p>

<p><strong>Please note: The bamout file cannot be generated when using <code class="code codeInline" spellcheck="false">-nt</code> or <code class="code codeInline" spellcheck="false">-nct</code>.</strong></p>

<h3>2. Generating the bamout for a single site or interval</h3>

<p>To generate the bamout file for a specific site or interval, just run HaplotypeCaller on the region around the site or interval of interest using the <code class="code codeInline" spellcheck="false">-L</code> argument to restrict the analysis to that region (adding about 500 bp on either side) and using the  <code class="code codeInline" spellcheck="false">-bamout</code> argument to specify the name of the bamout file that will be generated.</p>

<pre class="code codeBlock" spellcheck="false">java -jar GenomeAnalysisTK.jar -T HaplotypeCaller -R human_b37_20.fasta -I recalibrated.bam -o hc_variants.vcf -L 20:10255630-10255840 -bamout bamout.bam
</pre>

<p><em>If you were using any additional parameters in your original variant calling (including <code class="code codeInline" spellcheck="false">-ERC</code> and related arguments), make sure to include them in this command as well so that you can make an apples-to-apples comparison.</em></p>

<p>Then you open up both the original bam and the bamout file together in a genome browser such as IGV. On some test data from our favorite sample, NA12878, this is what you would see:</p>

<p><a rel="nofollow" href="https://us.v-cdn.net/5019796/uploads/FileUpload/1d/8f3640132b2107d3180a708deb6544.png"><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/1d/8f3640132b2107d3180a708deb6544.png" alt="image" class="embedImage-img importedEmbed-img"></img></a></p>

<p>You can see that the bamout file, on top, contains data only for the ActiveRegion that was within the analysis interval specified by <code class="code codeInline" spellcheck="false">-L</code>. The two blue reads represent the artificial haplotypes constructed by HaplotypeCaller (you may need to adjust your IGV settings to see the same thing on your machine).</p>

<p>You can see a whole group of reads neatly aligned, with an insertion in the middle. In comparison, the original data shown in the lower track has fewer reads with insertions, but has several reads with mismapped ends. This is a classic example of a site where realignment through reassembly has provided additional evidence for an indel, allowing HaplotypeCaller to call it confidently. In contrast, UnifiedGenotyper was not able to call this insertion confidently.</p>

<h3>3. Generating the bamout for multiple intervals or the whole genome</h3>

<p>Although we don't recommend doing this by default because it will cause slower performance and take up a lot of storage space, you can generate a bamout that contains many more intervals, or even covers the whole genome. To do so, just run the same command, but this time, pass your list of intervals to <code class="code codeInline" spellcheck="false">-L</code>, or simply omit it if you want the entire genome to be included.</p>

<pre class="code codeBlock" spellcheck="false">java -jar GenomeAnalysisTK.jar -T HaplotypeCaller -R human_b37_20.fasta -I recalibrated.bam -o hc_variants.vcf -bamout bamout.bam
</pre>

<p>This time, if you zoom out a bit in IGV, you will see multiple stacks of reads corresponding to the various ActiveRegions that were identified and processed.</p>

<p><a rel="nofollow" href="https://us.v-cdn.net/5019796/uploads/FileUpload/ee/28fc0d190a7342829a3a1965b7d414.png"><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/ee/28fc0d190a7342829a3a1965b7d414.png" alt="image" class="embedImage-img importedEmbed-img"></img></a></p>

<h3>4. Forcing an output in a region that is not covered in the bamout</h3>

<p>In some cases HaplotypeCaller does not complete processing on an ActiveRegion that it has started. This is typically because there is either almost no evidence of variation once the remapping has been done, or on the contrary, the region is very messy and there is too much complexity. In both cases, the program is designed to give up in order to avoid wasting time. This is a good thing most of the time, but it does mean that sometimes you will have no output in the bamout for the site you are trying to troubleshoot.</p>

<p>The good news is that in most cases it is possible to force HaplotypeCaller to go through with the full processing so that it will produce bamout output for your site of interest. To do so, simply add the flags <code class="code codeInline" spellcheck="false">-forceActive</code> and <code class="code codeInline" spellcheck="false">-disableOptimizations</code> to your command line, in addition to the <code class="code codeInline" spellcheck="false">-bamout</code> argument of course.</p>

<pre class="code codeBlock" spellcheck="false">java -jar GenomeAnalysisTK.jar -T HaplotypeCaller -R human_b37_20.fasta -I recalibrated.bam -L 20:10371667-10375021 -o hc_forced.vcf -bamout force_bamout.bam -forceActive -disableOptimizations 
</pre>

<p>In this other region, you can see that the original mapping (middle track) was a bit messy with some possible evidence of variation, and in fact UnifiedGenotyper called a SNP in this region (top variant track). But HaplotypeCaller did not call the SNP, and did not output anything in our first bamout file (top read track). When you force an output in that region using the two new flags, you see in the forced bamout (bottom read track) that the remapped data is a lot cleaner and the evidence for variation is essentially gone.</p>

<p><a rel="nofollow" href="https://us.v-cdn.net/5019796/uploads/FileUpload/87/242a92640f46be4d21ee9ea12f562f.png"><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/87/242a92640f46be4d21ee9ea12f562f.png" alt="image" class="embedImage-img importedEmbed-img"></img></a></p>

<p>It is also possible to force an ActiveRegion to be triggered at specific intervals; see the <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_haplotypecaller_HaplotypeCaller.php">HaplotypeCaller tool docs</a> for more details on how this is done.</p>
