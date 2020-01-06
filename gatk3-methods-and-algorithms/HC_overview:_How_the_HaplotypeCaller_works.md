## HC overview: How the HaplotypeCaller works

By Geraldine_VdAuwera

<p>This document describes the methods involved in variant calling as performed by the HaplotypeCaller. Please note that we are still working on producing supporting figures to help explain the sometimes complex operations involved.</p>

<h3>Overview</h3>

<p>The core operations performed by HaplotypeCaller can be grouped into these major steps:</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/a4/5ac06fc8af4b1b0c474f03e45f9017.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p><strong>1. Define active regions.</strong> The program determines which regions of the genome it needs to operate on, based on the presence of significant evidence for variation.</p>

<p><strong>2. Determine haplotypes by re-assembly of the active region.</strong> For each ActiveRegion, the program builds a De Bruijn-like graph to reassemble the ActiveRegion and identifies what are the possible haplotypes present in the data. The program then realigns each haplotype against the reference haplotype using the Smith-Waterman algorithm in order to identify potentially variant sites.</p>

<p><strong>3. Determine likelihoods of the haplotypes given the read data.</strong> For each ActiveRegion, the program performs a pairwise alignment of each read against each haplotype using the PairHMM algorithm. This produces a matrix of likelihoods of haplotypes given the read data. These likelihoods are then marginalized to obtain the likelihoods of alleles per read for each potentially variant site.</p>

<p><strong>4. Assign sample genotypes.</strong> For each potentially variant site, the program applies Bayes’ rule, using the likelihoods of alleles given the read data to calculate the posterior likelihoods of each genotype per sample given the read data observed for that sample. The most likely genotype is then assigned to the sample.</p>

<hr></hr><h3>1. Define active regions</h3>

<p>In this first step, the program traverses the sequencing data to identify regions of the genomes in which the samples being analyzed show substantial evidence of variation relative to the reference. The resulting areas are defined as “active regions”, and will be passed on to the next step. Areas that do not show any variation beyond the expected levels of background noise will be skipped in the next step. This aims to accelerate the analysis by not wasting time performing reassembly on regions that are identical to the reference anyway.</p>

<p>To define these active regions, the program operates in three phases. First, it computes an <strong>activity score</strong> for each individual genome position, yielding the <strong>raw activity profile</strong>, which is a wave function of activity per position. Then, it applies a smoothing algorithm to the raw profile, which is essentially a sort of averaging process, to yield the actual <strong>activity profile</strong>. Finally, it identifies local maxima where the activity profile curve rises above the preset activity threshold, and defines appropriate intervals to encompass the active profile within the preset size constraints. For more details on how the activity profile is computed and processed, as well as what options are available to modify the active region parameters, please see <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=4147">this method article</a>.</p>

<p>Note that the process for determining active region intervals is modified slightly when HaplotypeCaller is run in one of the special modes, e.g. the <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=4042">reference confidence mode</a> <a rel="nofollow" href="http://www.broadinstitute.org/gatk/gatkdocs/org_broadinstitute_sting_gatk_walkers_haplotypecaller_HaplotypeCaller.html#--emitRefConfidence">(<code class="code codeInline" spellcheck="false">-ERC GVCF</code> or <code class="code codeInline" spellcheck="false">ERC BP_RESOLUTION</code>)</a>, Genotype Given Alleles <a rel="nofollow" href="http://www.broadinstitute.org/gatk/gatkdocs/org_broadinstitute_sting_gatk_walkers_haplotypecaller_HaplotypeCaller.html#--genotyping_mode">(<code class="code codeInline" spellcheck="false">-gt_mode GENOTYPE_GIVEN_ALLELES</code>)</a> or when active regions are triggered using advanced arguments such as <a rel="nofollow" href="http://www.broadinstitute.org/gatk/gatkdocs/org_broadinstitute_sting_gatk_walkers_haplotypecaller_HaplotypeCaller.html#--useAllelesTrigger"><code class="code codeInline" spellcheck="false">-allelesTrigger</code></a>, <a rel="nofollow" href="http://www.broadinstitute.org/gatk/gatkdocs/org_broadinstitute_sting_gatk_walkers_haplotypecaller_HaplotypeCaller.html#--forceActive"><code class="code codeInline" spellcheck="false">--forceActive</code></a> or <a rel="nofollow" href="http://www.broadinstitute.org/gatk/gatkdocs/org_broadinstitute_sting_gatk_walkers_haplotypecaller_HaplotypeCaller.html#--activeRegionIn"><code class="code codeInline" spellcheck="false">--activeRegionIn</code></a>. This is covered in the method article referenced above.</p>

<p>Once this process is complete, the program applies a few post-processing steps to finalize the the active regions (see detailed doc above). The final output of this process is a list of intervals corresponding to the active regions which will be processed in the next step.</p>

<hr></hr><h3>2. Determine haplotypes by re-assembly of the active region.</h3>

<p>The goal of this step is to reconstruct the possible sequences of the real physical segments of DNA present in the original sample organism. To do this, the program goes through each active region and uses the input reads that mapped to that region to construct complete sequences covering its entire length, which are called haplotypes. This process will typically generate several different possible haplotypes for each active region due to:</p>

<ul><li>real diversity on polyploid (including CNV) or multi-sample data</li>
<li>possible allele combinations between variant sites that are not totally linked within the active region</li>
<li>sequencing and mapping errors</li>
</ul><p>In order to generate a list of possible haplotypes, the program first builds an assembly graph for the active region using the reference sequence as a template. Then, it takes each read in turn and attempts to match it to a segment of the graph. Whenever portions of a read do not match the local graph, the program adds new nodes to the graph to account for the mismatches. After this process has been repeated with many reads, it typically yields a complex graph with many possible paths. However, because the program keeps track of how many reads support each path segment, we can select only the most likely (well-supported) paths. These likely paths are then used to build the haplotype sequences which will be used for scoring and genotyping in the next step.</p>

<p>The assembly and haplotype determination procedure is described in full detail in <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=4146">this method article</a>.</p>

<p>Once the haplotypes have been determined, each one is realigned against the original reference sequence in order to identify potentially variant sites. This produces the set of sites that will be processed in the next step. A subset of these sites will eventually be emitted as variant calls to the output VCF.</p>

<hr></hr><h3>3. Evaluating the evidence for haplotypes and variant alleles</h3>

<p>Now that we have all these candidate haplotypes, we need to evaluate how much evidence there is in the data to support each one of them. So the program takes each individual read and aligns it against each haplotype in turn (including the reference haplotype) using the PairHMM algorithm, which takes into account the information we have about the quality of the data (i.e. the base quality scores and indel quality scores). This outputs a score for each read-haplotype pairing, expressing the likelihood of observing that read given that haplotype.</p>

<p>Those scores are then used to calculate out how much evidence there is for individual alleles at the candidate sites that were identified in the previous step. The process is called <strong>marginalization over alleles</strong> and produces the actual numbers that will finally be used to assign a genotype to the sample in the next step.</p>

<p>For further details on the pairHMM output and the marginalization process, see <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=4441">this document</a>.</p>

<hr></hr><h3>4. Assigning per-sample genotypes</h3>

<p>The previous step produced a table of per-read allele likelihoods for each candidate variant site under consideration. Now, all that remains to do is to evaluate those likelihoods in aggregate to determine what is the most likely genotype of the sample at each site. This is done by applying Bayes' theorem to calculate the likelihoods of each possible genotype, and selecting the most likely. This produces a genotype call as well as the calculation of various metrics that will be annotated in the output VCF if a variant call is emitted.</p>

<p>For further details on the genotyping calculations, see <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=4442">this document</a>.</p>

<p>This concludes the overview of how HaplotypeCaller works.</p>
