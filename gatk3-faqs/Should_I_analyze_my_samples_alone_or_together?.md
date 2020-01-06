## Should I analyze my samples alone or together?

By Geraldine_VdAuwera

<h3>Together is (almost always) better than alone</h3>

<p>We recommend performing variant discovery in a way that enables joint analysis of multiple samples, as laid out in our <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/best-practices">Best Practices</a> workflow. That workflow includes a joint analysis step that empowers variant discovery by providing the ability to leverage population-wide information from a cohort of multiple sample, allowing us to detect variants with great sensitivity and genotype samples as accurately as possible. Our workflow recommendations provide a way to do this in a way that is scalable and allows incremental processing of the sequencing data.</p>

<p>The key point is that you don’t actually have to call variants on all your samples together to perform a joint analysis. We have developed a workflow that allows us to decouple the initial identification of potential variant sites (ie variant calling) from the genotyping step, which is the only part that really needs to be done jointly. Since GATK 3.0, you can use the HaplotypeCaller to call variants individually per-sample in <code class="code codeInline" spellcheck="false">-ERC GVCF</code> mode, followed by a joint genotyping step on all samples in the cohort, as described in <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=3893">this method article</a>. This achieves what we call incremental joint discovery, providing you with all the benefits of classic joint calling (as described below) without the drawbacks.</p>

<p><strong>Why "almost always"?</strong> Because some people have reported missing a small fraction of singletons (variants that are unique to individual samples) when using the new method. For most studies, this is an acceptable tradeoff (which is reduced by the availability of high quality sequencing data), but if you are very specifically looking for singletons, you may need to do some careful evaluation before committing to this method.</p>

<hr></hr><h3>Previously established cohort analysis strategies</h3>

<p>Until recently, three strategies were available for variant discovery in multiple samples:</p>

<p><strong>- single sample calling:</strong> sample BAMs are analyzed individually, and individual call sets are combined in a downstream processing step;<br><strong>- batch calling:</strong> sample BAMs are analyzed in separate batches, and batch call sets are merged in a downstream processing step;<br><strong>- joint calling:</strong> variants are called simultaneously across all sample BAMs, generating a single call set for the entire cohort.</p>

<p>The best of these, from the point of view of variant discovery, was joint calling, because it provided the following benefits:</p>

<h4>1. Clearer distinction between homozygous reference sites and sites with missing data</h4>

<p>Batch-calling does not output a genotype call at sites where no member in the batch has evidence for a variant; it is thus impossible to distinguish such sites from locations missing data. In contrast, joint calling emits genotype calls at every site where any individual in the call set has evidence for variation.</p>

<h4>2. Greater sensitivity for low-frequency variants</h4>

<p>By sharing information across all samples, joint calling makes it possible to “rescue” genotype calls at sites where a carrier has low coverage but other samples within the call set have a confident variant at that location. However this does not apply to singletons, which are unique to a single sample. To minimize the chance of missing singletons, we increase the cohort size -- so that singletons themselves have less chance of happening in the first place.</p>

<h4>3. Greater ability to filter out false positives</h4>

<p>The current approaches to variant filtering (such as VQSR) use statistical models that work better with large amounts of data. Of the three calling strategies above, only joint calling provides enough data for accurate error modeling and ensures that filtering is applied uniformly across all samples.</p>

<p><a rel="nofollow" href="https://us.v-cdn.net/5019796/uploads/FileUpload/40/3d322e97441f1918626854d56c2574.png"><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/40/3d322e97441f1918626854d56c2574.png" alt="image" class="embedImage-img importedEmbed-img"></img></a></p>

<p><strong>Figure 1:</strong> <em>Power of joint calling in finding mutations at low coverage sites. The variant allele is present in only two of the N samples, in both cases with such low coverage that the variant is not callable when processed separately. Joint calling allows evidence to be accumulated over all samples and renders the variant callable. (right) Importance of joint calling to square off the genotype matrix, using an example of two disease-relevant variants. Neither sample will have records in a variants-only output file, for different reasons: the first sample is homozygous reference while the second sample has no data. However, merging the results from single sample calling will incorrectly treat both of these samples identically as being non-informative.</em></p>

<hr></hr><h3>Drawbacks of traditional joint calling (all steps performed multi-sample)</h3>

<p>There are two major problems with the joint calling strategy.</p>

<p><strong>- Scaling &amp; infrastructure</strong><br>
Joint calling scales very badly -- the calculations involved in variant calling (especially by methods like the HaplotypeCaller’s) become exponentially more computationally costly as you add samples to the cohort. If you don't have a lot of compute available, you run into limitations pretty quickly. Even here at Broad where we have fairly ridiculous amounts of compute available, we can't brute-force our way through the numbers for the larger cohort sizes that we're called on to handle.</p>

<p><strong>- The N+1 problem</strong><br>
When you’re getting a large-ish number of samples sequenced (especially clinical samples), you typically get them in small batches over an extended period of time, and you analyze each batch as it comes in (whether it’s because the analysis is time-sensitive or your PI is breathing down your back). But that’s not joint calling, that’s batch calling, and it doesn’t give you the same significant gains that joint calling can give you. Unfortunately the joint calling approach doesn’t allow for incremental analysis -- every time you get even one new sample sequence, you have to re-call all samples from scratch.</p>

<h4>Both of these problems are solved by the single-sample calling + joint genotyping workflow.</h4>
