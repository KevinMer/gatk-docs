## Likelihoods and Probabilities

By dekling

<p>There are several instances in the GATK documentation where you will encounter the terms "likelihood" and "probability", because key tools in the variant discovery workflow rely heavily on Bayesian statistics. For example, the <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_haplotypecaller_HaplotypeCaller.php">HaplotypeCaller</a>, our most prominent germline SNP and indel caller, uses Bayesian statistics to <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/article?id=4442">determine genotypes</a>.</p>

<h4>So what do likelihood and probability mean and how are they related to each other in the Bayesian context?</h4>

<p>In Bayesian statistics (as opposed to <a rel="nofollow" href="https://xkcd.com/1132/">frequentist statistics</a>), we are typically trying to evaluate the <a rel="nofollow" href="https://en.wikipedia.org/wiki/Posterior_probability">posterior probability</a> of a hypothesis (H) based on a series of observations (data, D).</p>

<p><strong>Bayes' rule</strong> states that</p>

<p>$${P(H|D)}=\frac{P(H)P(D|H)}{P(D)}$$</p>

<p>where the bit we care about most, <strong>P(D|H)</strong>, is the <strong>probability of observing D given the hypothesis H</strong>. This can also be formulated as <strong>L(H|D)</strong>, i.e. the <strong>likelihood of the hypothesis H given the observation D</strong>:</p>

<p>$$P(D|H)=L(H|D)$$</p>

<p>We use the term <strong>likelihood</strong> instead of <strong>probability</strong> to describe the term on the right because we cannot calculate a meaningful probability distribution on a hypothesis, which by definition is binary (it will either be true or false) -- but we <em>can</em> determine the likelihood that a hypothesis is true or false given a set of observations.  For a more detailed explanation of these concepts, please see the following lesson (<a href="http://ocw.mit.edu/courses/mathematics/18-05-introduction-to-probability-and-statistics-spring-2014/readings/MIT18_05S14_Reading11.pdf" rel="nofollow">http://ocw.mit.edu/courses/mathematics/18-05-introduction-to-probability-and-statistics-spring-2014/readings/MIT18_05S14_Reading11.pdf</a>).</p>

<p>Now you may wonder, what about the posterior probability P(H|D) that we eventually calculate through Bayes' rule? Isn't that a "probability of a hypothesis"? Well yes; in Bayesian statistics, we <em>can</em> calculate a <em>posterior</em> probability distribution on a hypothesis, because its probability distribution is <em>relative</em> to all of the other competing hypotheses (<a href="http://www.smbc-comics.com/index.php?id=4127" rel="nofollow">http://www.smbc-comics.com/index.php?id=4127</a>). Tadaa.</p>

<p>See <a rel="nofollow" href="https://www.broadinstitute.org/gatk/guide/article?id=4442">this HaplotypeCaller doc article</a> for a worked out explanation of how we calculate and use genotype likelihoods in germline variant calling.</p>

<p>So always remember this, if nothing else: the terms likelihood and probability are <em>not</em> interchangeable in the Bayesian context, even though they are often used interchangeably in common English.</p>

<p>A special thanks to Jon M. Bloom PhD (MIT) for his assistance in the preparation of this article.</p>
