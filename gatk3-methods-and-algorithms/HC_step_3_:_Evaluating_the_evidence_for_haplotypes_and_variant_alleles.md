## HC step 3 : Evaluating the evidence for haplotypes and variant alleles

By Sheila

<p>This document describes the procedure used by HaplotypeCaller to evaluate the evidence for variant alleles based on candidate haplotypes determined in <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=4146">the previous step</a> for a given ActiveRegion. For more context information on how this fits into the overall HaplotypeCaller method, please see the more general <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=4148">HaplotypeCaller documentation</a>.</p>

<h3>Overview</h3>

<p>The previous step produced a list of candidate haplotypes for each ActiveRegion, as well as a list of candidate variant sites borne by the non-reference haplotypes. Now, we need to evaluate how much evidence there is in the data to support each haplotype. This is done by aligning each sequence read to each haplotype using the PairHMM algorithm, which produces per-read likelihoods for each haplotype. From that, we'll be able to derive how much evidence there is in the data to support each variant allele at the candidate sites, and that produces the actual numbers that will finally be used to assign a genotype to the sample.</p>

<hr></hr><h3>1. Evaluating the evidence for each candidate haplotype</h3>

<p>We originally obtained our list of haplotypes for the ActiveRegion by constructing an assembly graph and selecting the most likely paths in the graph by counting the number of supporting reads for each path. That was a fairly naive evaluation of the evidence, done over all reads in aggregate, and was only meant to serve as a preliminary filter to whittle down the number of possible combinations that we're going to look at in this next step.</p>

<p>Now we want to do a much more thorough evaluation of how much evidence we have for each haplotype. So we're going to take each individual read and align it against each haplotype in turn (including the reference haplotype) using the PairHMM algorithm (see Durbin <em>et al.</em>, 1998). If you're not familiar with PairHMM, it's a lot like the BLAST algorithm, in that it's a pairwise alignment method that uses a <a rel="nofollow" href="http://www.fejes.ca/easyhmm.html">Hidden Markov Model (HMM)</a> and produces a likelihood score. In this use of the PairHMM, the output score expresses the likelihood of observing the read given the haplotype by taking into account the information we have about the quality of the data (i.e. the base quality scores and indel quality scores). <strong>Note: If reads from a pair overlap at a site and they have the same base, the base quality is capped at Q20 for both reads (Q20 is half the expected PCR error rate). If they do not agree, we set both base qualities to Q0.</strong></p>

<p>This produces a big table of likelihoods where the columns are haplotypes and the rows are individual sequence reads. <strong>(example figure TBD)</strong></p>

<p>The table essentially represents how much supporting evidence there is for each haplotype (including the reference), itemized by read.</p>

<hr></hr><h3>2. Evaluating the evidence for each candidate site and corresponding alleles</h3>

<p>Having per-read likelihoods for entire haplotypes is great, but ultimately we want to know how much evidence there is for individual alleles at the candidate sites that we identified in the previous step. To find out, we take the per-read likelihoods of the haplotypes and <strong>marginalize them over alleles</strong>, which produces per-read likelihoods for each allele at a given site. In practice, this means that for each candidate site, we're going to decide how much support each read contributes for each allele, based on the per-read haplotype likelihoods that were produced by the PairHMM.</p>

<p>This may sound complicated, but the procedure is actually very simple -- there is no real calculation involved, just cherry-picking appropriate values from the table of per-read likelihoods of haplotypes into a new table that will contain per-read likelihoods of alleles. This is how it happens. For a given site, we list all the alleles observed in the data (including the reference allele). Then, for each read, we look at the haplotypes that support each allele; we select the haplotype that has the highest likelihood for that read, and we write that likelihood in the new table. And that's it! For a given allele, the total likelihood will be the product of all the per-read likelihoods. <strong>(example fig TBD)</strong></p>

<p>At the end of this step, sites where there is sufficient evidence for at least one of the variant alleles considered will be called variant, and a genotype will be assigned to the sample in the next (final) step.</p>
