## Understanding and adapting the generic hard-filtering recommendations

By Sheila

<p>This document aims to provide insight into the logic of the <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/article?id=3225">generic hard-filtering recommendations</a> that we provide as a substitute for VQSR. Hopefully it will also serve as a guide for adapting these recommendations or developing new filters that are appropriate for datasets that diverge significantly from what we usually work with.</p>

<hr></hr><h3>Introduction</h3>

<p>Hard-filtering consists of choosing specific thresholds for one or more annotations and throwing out any variants that have annotation values above or below the set thresholds. By annotations, we mean properties or <em>statistics</em> that describe for each variant <em>e.g.</em> what the sequence context is like around the variant site, how many reads covered it, how many reads covered each allele, what proportion of reads were in forward vs reverse orientation, and so on.</p>

<p>The problem with this approach is that it is very limiting because it forces you to look at each annotation dimension individually, and you end up throwing out good variants just because one of their annotations looks bad, or keeping bad variants in order to keep those good variants.</p>

<p>In contrast, VQSR is more powerful because it uses machine-learning algorithms to learn from the data what are the annotation profiles of good variants (true positives) and of bad variants (false positives) in a particular dataset. This empowers you to pull out variants based on how they cluster together along different dimensions, and liberates you to a large extent from the linear tyranny of single-dimension thresholds.</p>

<p>Unfortunately this method requires a large number of variants and well-curated known variant resources. For those of you working with small gene panels or with non-model organisms, this is a deal-breaker, and you have to fall back on hard-filtering.</p>

<hr></hr><h3>Outline</h3>

<p>In this article, we illustrate how the generic hard-filtering recommendations we provide relate to the distribution of annotation values we typically see in callsets produced by our variant calling tools, and how this in turn relates to the underlying physical properties of the sequence data.</p>

<p>We also use results from VQSR filtering (which we take as ground truth in this context) to highlight the limitations of hard-filtering.</p>

<p>We do this in turn for each of five annotations that are highly informative among the recommended annotations: QD, FS, MQ, MQRankSum and ReadPosRankSum. The same principles can be applied to most other annotations produced by GATK tools.</p>

<hr></hr><h3>Overview of data and methods</h3>

<h4>Origin of the dataset</h4>

<p>We called variants on a whole genome trio (samples NA12878, NA12891, NA12892, previously pre-processed) using HaplotypeCaller in GVCF mode, yielding a gVCF file for each sample. We then joint-genotyped the gVCFs using GenotypeGVCF, yielding an unfiltered VCF callset for the trio. Finally, we ran VQSR on the trio VCF, yielding the filtered callset. We will be looking at the SNPs only.</p>

<h4>Plotting methods and interpretation notes</h4>

<p>All plots shown below are density plots generated using the ggplot2 library in R. On the x-axis are the annotation values, and on the y-axis are the density values. The area under the density plot gives you the probability of observing the annotation values. So, the entire area under all of the plots will be equal to 1. However, if you would like to know the probability of observing an annotation value between 0 and 1, you will have to take the area under the curve between 0 and 1.</p>

<p>In plain English, this means that the plots shows you, for a given set of variants, what is the distribution of their annotation values. The caveat is that when we're comparing two or more sets of variants on the same plot, we have to keep in mind that they may contain very different numbers of variants, so the amount of variants in a given part of the distribution is not directly comparable; only their <em>proportions</em> are comparable.</p>

<hr></hr><h3><a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_QualByDepth.php">QualByDepth (QD)</a></h3>

<p>This is the variant confidence (from the QUAL field) divided by the unfiltered depth of non-hom-ref samples. This annotation is intended to normalize the variant quality in order to avoid inflation caused when there is deep coverage. For filtering purposes it is better to use QD than either QUAL or DP directly.</p>

<p>The generic filtering recommendation for QD is to filter out variants with QD below 2. Why is that?</p>

<p>First, let’s look at the QD values distribution for unfiltered variants.  Notice the values can be anywhere from 0-40. There are two peaks where the majority of variants are (around QD = 12 and QD = 32). These two peaks correspond to variants that are mostly observed in heterozygous (het) versus mostly homozygous-variant (hom-var) states, respectively, in the called samples. This is because hom-var samples contribute twice as many reads supporting the variant than do het variants. We also see, to the left of the distribution, a "shoulder" of variants with QD hovering between 0 and 5.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/e7/e150bb10a34b3c98429d060b791880.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p><em>We expect to see a similar distribution profile in callsets generated from most types of high-throughput sequencing data, although values where the peaks form may vary.</em></p>

<p>Now, let’s look at the plot of QD values for variants that passed VQSR and those that failed VQSR. Red indicates the variants that failed VQSR, and blue (green?) the variants that passed VQSR.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/4e/367a856e3dfe1e016caa1a06a524b3.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>We see that the majority of variants filtered out correspond to that low-QD "shoulder" (remember that since this is a density plot, the y-axis indicates proportion, not number of variants); that is what we would filter out with the generic recommendation of the threshold value 2 for QD.</p>

<p>Notice however that VQSR has failed some variants that have a QD greater than 30! All those variants would have passed the hard filter threshold, but VQSR tells us that these variants looked artifactual in one or more other annotation dimensions. Conversely, although it is not obvious in the figure, we know that VQSR has passed some variants that have a QD less than 2, which hard filters would have eliminated from our callset.</p>

<hr></hr><h3><a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_FisherStrand.php">FisherStrand (FS)</a></h3>

<p>This is the Phred-scaled probability that there is strand bias at the site. Strand Bias tells us whether the alternate allele was seen more or less often on the forward or reverse strand than the reference allele. When there little to no strand bias at the site, the FS value will be close to 0.</p>

<p><strong>Note:</strong> SB, SOR and FS are related but not the same! They all measure strand bias  (a type of sequencing bias in which one DNA strand is favored over the other, which can result in incorrect evaluation of the amount of evidence observed for one allele vs. the other) in different ways. SB gives the raw counts of reads supporting each allele on the forward and reverse strand. FS is the result of using those counts in a Fisher's Exact Test. SOR is a related annotation that applies a different statistical test (using the SB counts) that is better for high coverage data.</p>

<p>Let’s look at the FS values for the unfiltered variants. The FS values have a very wide range; we made the x-axis log-scaled so the distribution is easier to see. Notice most variants have an FS value less than 10, and almost all variants have an FS value less than 100. However, there are indeed some variants with a value close to 400.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/e7/8a8d6c15eefbdad7c7757c1a731f3a.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>The plot below shows FS values for variants that passed VQSR and failed VQSR.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/06/b1f0fc767a9976f7e059095813afee.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>Notice most of the variants that fail have an FS value greater than 55. Our hard filtering recommendations tell us to fail variants with an FS value greater than 60. Notice that although we are able to remove many false positives by removing variants with FS greater than 60, we still keep many false positive variants. If we move the threshold to a lower value, we risk losing true positive variants.</p>

<hr></hr><h3><a rel="nofollow" href="https://software.broadinstitute.org/gatk/gatkdocs/org_broadinstitute_gatk_tools_walkers_annotator_StrandOddsRatio.php">StrandOddsRatio (SOR)</a></h3>

<p>This is another way to estimate strand bias using a test similar to the symmetric odds ratio test. SOR was created because FS tends to penalize variants that occur at the ends of exons. Reads at the ends of exons  tend to only be covered by reads in one direction and FS gives those variants a bad score. SOR will take into account the ratios of reads that cover both alleles.</p>

<p>Let’s look at the SOR values for the unfiltered variants. The SOR values range from 0 to greater than 9. Notice most variants have an SOR value less than 3, and almost all variants have an SOR value less than 9. However, there is a long tail of variants with a value greater than 9.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/78/d85211fc92ef471f580414f3b41695.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>The plot below shows SOR values for variants that passed VQSR and failed VQSR.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/3a/daf7279b526be324e3241933ac0622.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>Notice most of the variants that have an SOR value greater than 3 fail the VQSR filter. Although there is a non-negligible population of variants with an SOR value less than 3 that failed VQSR, our hard filtering recommendation of failing variants with an SOR value greater than 3 will at least remove the long tail of variants that show fairly clear bias according to the SOR test.</p>

<hr></hr><h3><a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_RMSMappingQuality.php">RMSMappingQuality (MQ)</a></h3>

<p>This is the root mean square mapping quality over all the reads at the site. Instead of the average mapping quality of the site, this annotation gives the square root of the average of the squares of the mapping qualities at the site. It is meant to include the standard deviation of the mapping qualities. Including the standard deviation allows us to include the variation in the dataset. A low standard deviation means the values are all close to the mean, whereas a high standard deviation means the values are all far from the mean.When the mapping qualities are good at a site, the MQ will be around 60.</p>

<p>Now let’s check out the graph of MQ values for the unfiltered variants. Notice the very large peak around MQ = 60. Our recommendation is to fail any variant with an MQ value less than 40.0. You may argue that hard filtering any variant with an MQ value less than 50 is fine as well. This brings up an excellent point that our hard filtering recommendations are meant to be very lenient. We prefer to keep all potentially decent variants rather than get rid of a few bad variants.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/2f/e30627c1bb27a5937a41991fcec031.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>Let’s look at the VQSR pass vs fail variants. At first glance, it seems like VQSR has passed the variants in the high peak and failed any variants not in the peak.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/5e/5d92739850c9f2114d77813aca0660.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>It is hard to tell which variants passed and failed, so let’s zoom in and see what exactly is happening.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/3c/0fa58f6a4a8bc35a1187c7118b3783.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>The plot above shows the x-axis from 59-61. Notice the variants in blue (the ones that passed) all have MQ around 60. However, some variants in red (the ones that failed) also have an MQ around 60.</p>

<hr></hr><h3><a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_MappingQualityRankSumTest.php">MappingQualityRankSumTest (MQRankSum)</a></h3>

<p>This is the u-based z-approximation from the Rank Sum Test for mapping qualities. It compares the mapping qualities of the reads supporting the reference allele and the alternate allele. A positive value means the mapping qualities of the reads supporting the alternate allele are higher than those supporting the reference allele; a negative value indicates the mapping qualities of the reference allele are higher than those supporting the alternate allele. A value close to zero is best and indicates little difference between the mapping qualities.</p>

<p>Next, let’s look at the distribution of values for MQRankSum in the unfiltered variants. Notice the values range from approximately -10.5 to 6.5. Our hard filter threshold is -12.5. There are no variants in this dataset that have MQRankSum less than -10.5! In this case, hard filtering would not fail any variants based on MQRankSum. Remember, our hard filtering recommendations are meant to be very lenient. If you do plot your annotation values for your samples and find none of your variants have MQRankSum less than -12.5, you may want to refine your hard filters. Our recommendations are indeed recommendations that you the scientist will want to refine yourself.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/43/e18f4c069b972f6894f51312234dca.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>Looking at the plot of pass VQSR vs fail VQSR variants, we see the variants with an MQRankSum value less than -2.5 fail VQSR. However, the region between -2.5 to 2.5 contains both pass and fail variants. Are you noticing a trend here? It is very difficult to pick a threshold for hard filtering. If we pick -2.5 as our hard filtering threshold, we still have many variants that fail VQSR in our dataset. If we try to get rid of those variants, we will lose some good variants as well. It is up to you to decide how many false positives you would like to remove from your dataset vs how many true positives you would like to keep and adjust your threshold based on that.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/b6/1eb43e2bf19557ea38eb9520b61556.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<hr></hr><h3><a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_ReadPosRankSumTest.php">ReadPosRankSumTest (ReadPosRankSum)</a></h3>

<p>This is the u-based z-approximation from the Rank Sum Test for site position within reads. It compares whether the positions of the reference and alternate alleles are different within the reads. Seeing an allele only near the ends of reads is indicative of error, because that is where sequencers tend to make the most errors. A negative value indicates that the alternate allele is found at the ends of reads more often than the reference allele; a positive value indicates that the reference allele is found at the ends of reads more often than the alternate allele. A value close to zero is best because it indicates there is little difference between the positions of the reference and alternate alleles in the reads.</p>

<p>The last annotation we will look at is ReadPosRankSum. Notice the values fall mostly between -4 and 4. Our hard filtering threshold removes any variant with a ReadPosRankSum value less than -8.0. Again, there are no variants in this dataset that have a ReadPosRankSum value less than -8.0, but some datasets might. If you plot your variant annotations and find there are no variants that have a value less than or greater than one of our recommended cutoffs, you will have to refine them yourself based on your annotation plots.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/b2/7b0db901dcad3238b7866eed158265.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>Looking at the VQSR pass vs fail variants, we can see VQSR has failed variants with ReadPosRankSum values less than -1.0 and greater than 3.5. However, notice VQSR has failed some variants that have values that pass VQSR.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/f0/92bfb5089d4afdc3330732718b80d6.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>
