## Variant Quality Score Recalibration (VQSR)

By rpoplin

<p>This document describes what Variant Quality Score Recalibration (VQSR) is designed to do, and outlines how it works under the hood. The first section is a high-level overview aimed at non-specialists. Additional technical details are provided below.</p>

<p>For command-line examples and recommendations on what specific resource datasets and arguments to use for VQSR, please see <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=1259">this FAQ article</a>. See the <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_variantrecalibration_VariantRecalibrator.php">VariantRecalibrator tool doc</a> and the <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_variantrecalibration_ApplyRecalibration.php">ApplyRecalibration tool doc</a> for a complete description of available command line arguments.</p>

<p>As a complement to this document, we encourage you to watch the workshop videos available in the <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/presentations">Presentations section</a>.</p>

<hr></hr><h2>High-level overview</h2>

<p>VQSR stands for “variant quality score recalibration”, which is a bad name because it’s not <em>re</em>-calibrating variant quality scores at all; it is calculating a new quality score that is supposedly super well calibrated (unlike the variant QUAL score which is a hot mess) called the VQSLOD (for variant quality score log-odds). I know this probably sounds like gibberish, stay with me. The purpose of this new score is to enable variant filtering in a way that allows analysts to balance sensitivity (trying to discover all the real variants) and specificity (trying to limit the false positives that creep in when filters get too lenient) as finely as possible.</p>

<p>The basic, traditional way of filtering variants is to look at various annotations (context statistics) that describe e.g. what the sequence context is like around the variant site, how many reads covered it, how many reads covered each allele, what proportion of reads were in forward vs reverse orientation; things like that -- then choose threshold values and throw out any variants that have annotation values above or below the set thresholds. The problem with this approach is that it is very limiting because it forces you to look at each annotation dimension individually, and you end up throwing out good variants just because one of their annotations looks bad, or keeping bad variants in order to keep those good variants.</p>

<p>The VQSR method, in a nutshell, uses machine learning algorithms to learn from each dataset what is the annotation profile of good variants vs. bad variants, and does so in a way that integrates information from multiple dimensions (like, 5 to 8, typically). The cool thing is that this allows us to pick out clusters of variants in a way that frees us from the traditional binary choice of “is this variant above or below the threshold for this annotation?”</p>

<p>Let’s do a quick mental visualization exercise (pending an actual figure to illustrate this), in two dimensions because our puny human brains work best at that level. Imagine a topographical map of a mountain range, with North-South and East-West axes standing in for two variant annotation scales. Your job is to define a subset of territory that contains mostly mountain peaks, and as few lowlands as possible. Traditional hard-filtering forces you to set a single longitude cutoff and a single latitude cutoff, resulting in one rectangular quadrant of the map being selected, and all the rest being greyed out. It’s about as subtle as a sledgehammer and forces you to make a lot of compromises. VQSR allows you to select contour lines around the peaks and decide how low or how high you want to go to include or exclude territory within your subset.</p>

<p>How this is achieved is another can of worms. The key point is that we use known, highly validated variant resources (omni, 1000 Genomes, hapmap) to select a subset of variants within our callset that we’re really confident are probably true positives (that’s the training set). We look at the annotation profiles of those variants (in our own data!), and we from that we learn some rules about how to recognize good variants. We do something similar for bad variants as well. Then we apply the rules we learned to all of the sites, which (through some magical hand-waving) yields a single score for each variant that describes how likely it is based on all the examined dimensions. In our map analogy this is the equivalent of determining on which contour line the variant sits. Finally, we pick a threshold value <strong>indirectly</strong> by asking the question “what score do I need to choose so that e.g. 99% of the variants in my callset that are also in hapmap will be selected?”. This is called the target sensitivity. We can twist that dial in either direction depending on what is more important for our project, sensitivity or specificity.</p>

<hr></hr><h2></h2>

<h2>Technical overview</h2>

<p>The purpose of variant recalibration is to assign a well-calibrated probability to each variant call in a call set. This enables you to generate highly accurate call sets by filtering based on this single estimate for the accuracy of each call.</p>

<p>The approach taken by variant quality score recalibration is to develop a continuous, covarying estimate of the relationship between SNP call annotations (QD, SB, HaplotypeScore, HRun, for example) and the the probability that a SNP is a true genetic variant versus a sequencing or data processing artifact. This model is determined adaptively based on "true sites" provided as input (typically HapMap 3 sites and those sites found to be polymorphic on the Omni 2.5M SNP chip array, for humans). This adaptive error model can then be applied to both known and novel variation discovered in the call set of interest to evaluate the probability that each call is real. The score that gets added to the INFO field of each variant is called the VQSLOD. It is the log odds ratio of being a true variant versus being false under the trained Gaussian mixture model.</p>

<p>The variant recalibrator contrastively evaluates variants in a two step process, each performed by a distinct tool:</p>

<ul><li><p><em>VariantRecalibrator</em><br>
Create a Gaussian mixture model by looking at the annotations values over a high quality subset of the input call set and then evaluate all input variants. This step produces a recalibration file.</p></li>
<li><p><em>ApplyRecalibration</em><br>
Apply the model parameters to each variant in input VCF files producing a recalibrated VCF file in which each variant is annotated with its VQSLOD value. In addition, this step will filter the calls based on this new lod score by adding lines to the FILTER column for variants that don't meet the specified lod threshold.</p></li>
</ul><p>Please see the <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=2805">VQSR tutorial</a> for step-by-step instructions on running these tools.</p>

<hr></hr><h3>How VariantRecalibrator works in a nutshell</h3>

<p>The tool takes the overlap of the training/truth resource sets and of your callset. It models the distribution of these variants relative to the annotations you specified, and attempts to group them into clusters. Then it uses the clustering to assign VQSLOD scores to all variants. Variants that are closer to the heart of a cluster will get a higher score than variants that are outliers.</p>

<hr></hr><h3>How ApplyRecalibration works in a nutshell</h3>

<p>During the first part of the recalibration process, variants in your callset were given a score called VQSLOD. At the same time, variants in your training sets were also ranked by VQSLOD. When you specify a tranche sensitivity threshold with ApplyRecalibration, expressed as a percentage (e.g. 99.9%), what happens is that the program looks at what is the VQSLOD value above which 99.9% of the variants in the training callset are included. It then takes that value of VQSLOD and uses it as a threshold to filter your variants. Variants that are above the threshold pass the filter, so the FILTER field will contain PASS. Variants that are below the threshold will be filtered out; they will be written to the output file, but in the FILTER field they will have the name of the tranche they belonged to. So VQSRTrancheSNP99.90to100.00 means that the variant was in the range of VQSLODs corresponding to the remaining 0.1% of the training set, which are basically considered false positives.</p>

<hr></hr><h3>Interpretation of the Gaussian mixture model plots</h3>

<p>The variant recalibration step fits a Gaussian mixture model to the contextual annotations given to each variant.  By fitting this probability model to the training variants (variants considered to be true-positives), a probability can be assigned to the putative novel variants (some of which will be true-positives, some of which will be false-positives).  It is useful for users to see how the probability model was fit to their data. Therefore a modeling report is automatically generated each time VariantRecalibrator is run (in the above command line the report will appear as path/to/output.plots.R.pdf). For every pair-wise combination of annotations used in modeling, a 2D projection of the Gaussian mixture model is shown.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/7a/205c348ce50cb46dd94a8185de25ef.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>The figure shows one page of an example Gaussian mixture model report that is automatically generated by the VQSR from the example HiSeq call set. This page shows the 2D projection of mapping quality rank sum test versus Haplotype score by marginalizing over the other annotation dimensions in the model.</p>

<p>In each page there are four panels which show different ways of looking at the 2D projection of the model. The upper left panel shows the probability density function that was fit to the data. The 2D projection was created by marginalizing over the other annotation dimensions in the model via random sampling. Green areas show locations in the space that are indicative of being high quality while red areas show the lowest probability areas. In general putative SNPs that fall in the red regions will be filtered out of the recalibrated call set.</p>

<p>The remaining three panels give scatter plots in which each SNP is plotted in the two annotation dimensions as points in a point cloud. The scale for each dimension is in normalized units. The data for the three panels is the same but the points are colored in different ways to highlight different aspects of the data. In the upper right panel SNPs are colored black and red to show which SNPs are retained and filtered, respectively, by applying the VQSR procedure. The red SNPs didn't meet the given truth sensitivity threshold and so are filtered out of the call set. The lower left panel colors SNPs green, grey, and purple to give a sense of the distribution of the variants used to train the model. The green SNPs are those which were found in the training sets passed into the VariantRecalibrator step, while the purple SNPs are those which were found to be furthest away from the learned Gaussians and thus given the lowest probability of being true. Finally, the lower right panel colors each SNP by their known/novel status with blue being the known SNPs and red being the novel SNPs. Here the idea is to see if the annotation dimensions provide a clear separation between the known SNPs (most of which are true) and the novel SNPs (most of which are false).</p>

<p>An example of good clustering for SNP calls from the tutorial dataset is shown to the right. The plot shows that the training data forms a distinct cluster at low values for each of the two statistics shown (haplotype score and mapping quality bias). As the SNPs fall off the distribution in either one or both of the dimensions they are assigned a lower probability (that is, move into the red region of the model's PDF) and are filtered out. This makes sense as not only do higher values of HaplotypeScore indicate a lower chance of the data being explained by only two haplotypes but also higher values for mapping quality bias indicate more evidence of bias between the reference bases and the alternative bases. The model has captured our intuition that this area of the distribution is highly enriched for machine artifacts and putative variants here should be filtered out!</p>

<hr></hr><h3>Tranches and the tranche plot</h3>

<p>The recalibrated variant quality score provides a continuous estimate of the probability that each variant is true, allowing one to partition the call sets into quality tranches. The main purpose of the tranches is to establish thresholds within your data that correspond to certain levels of sensitivity relative to the truth sets. The idea is that with well calibrated variant quality scores, you can generate call sets in which each variant doesn't have to have a hard answer as to whether it is in or out of the set. If a very high accuracy call set is desired then one can use the highest tranche, but if a larger, more complete call set is a higher priority than one can dip down into lower and lower tranches. These tranches are applied to the output VCF file using the FILTER field. In this way you can choose to use some of the filtered records or only use the PASSing records.</p>

<p>The first tranche (90) which has the lowest value of truth sensitivity but the highest value of novel Ti/Tv, is exceedingly specific but less sensitive. Each subsequent tranche in turn introduces additional true positive calls along with a growing number of false positive calls. Downstream applications can select in a principled way more specific or more sensitive call sets or incorporate directly the recalibrated quality scores to avoid entirely the need to analyze only a fixed subset of calls but rather weight individual variant calls by their probability of being real.  An example tranche plot, automatically generated by the VariantRecalibrator walker, is shown below.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/b6/ef1c4b5fe263e3a24fea6848776cd8.jpeg" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>This is an example of a tranches plot generated for a HiSeq call set. The x-axis gives the number of novel variants called while the y-axis shows two quality metrics -- novel transition to transversion ratio and the overall truth sensitivity.</p>

<p>Note that the tranches plot is not applicable for indels and will not be generated when the tool is run in INDEL mode.</p>

<hr></hr><h3>Ti/Tv-free recalibration</h3>

<p>We use a Ti/Tv-free approach to variant quality score recalibration. This approach requires an additional truth data set, and cuts the VQSLOD at given sensitivities to the truth set.  It has several advantages over the Ti/Tv-targeted approach:</p>

<ul><li>The truth sensitivity (TS) approach gives you back the novel Ti/Tv as a QC metric</li>
<li>The truth sensitivity (TS) approach is conceptual cleaner than deciding on a novel Ti/Tv target for your dataset</li>
<li>The TS approach is easier to explain and defend, as saying "I took called variants until I found 99% of my known variable sites" is easier than "I took variants until I dropped my novel Ti/Tv ratio to 2.07"</li>
</ul><p>We have used hapmap 3.3 sites as the truth set (genotypes_r27_nr.b37_fwd.vcf), but other sets of high-quality (~99% truly variable in the population) sets of sites should work just as well. In our experience, with HapMap, 99% is a good threshold, as the remaining 1% of sites often exhibit unusual features like being close to indels or are actually MNPs, and so receive a low VQSLOD score.<br>
Note that the expected Ti/Tv is still an available argument but it is only used for display purposes.</p>

<hr></hr><h3>Finally, a couple of Frequently Asked Questions</h3>

<h4>- Can I use the variant quality score recalibrator with my small sequencing experiment?</h4>

<p>This tool is expecting thousands of variant sites in order to achieve decent modeling with the Gaussian mixture model. Whole exome call sets work well, but anything smaller than that scale might run into difficulties.</p>

<p>One piece of advice is to turn down the number of Gaussians used during training. This can be accomplished by adding <code class="code codeInline" spellcheck="false">--maxGaussians 4</code> to your command line.</p>

<p><code class="code codeInline" spellcheck="false">maxGaussians</code> is the maximum number of different "clusters" (=Gaussians) of variants the program is "allowed" to try to identify. Lowering this number forces the program to group variants into a smaller number of clusters, which means there will be more variants in each cluster -- hopefully enough to satisfy the statistical requirements. Of course, this decreases the level of discrimination that you can achieve between variant profiles/error modes. It's all about trade-offs; and unfortunately if you don't have a lot of variants you can't afford to be very demanding in terms of resolution.</p>

<h4>- Why don't all the plots get generated for me?</h4>

<p>The most common problem related to this is not having Rscript accessible in your environment path. Rscript is the command line version of <a rel="nofollow" href="http://cran.r-project.org/">R</a> that gets installed right alongside. We also make use of the <a rel="nofollow" href="http://ggplot2.org/">ggplot2 library</a> so please be sure to install that package as well. See the Common Problems section of the Guide for more details.</p>
