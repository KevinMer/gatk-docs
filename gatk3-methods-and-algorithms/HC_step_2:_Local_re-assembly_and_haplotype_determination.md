## HC step 2: Local re-assembly and haplotype determination

By Geraldine_VdAuwera

<p>This document details the procedure used by HaplotypeCaller to re-assemble read data and determine candidate haplotypes as a prelude to variant calling. For more context information on how this fits into the overall HaplotypeCaller method, please see the more general <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=4148">HaplotypeCaller documentation</a>.</p>

<p><em>Note that we are still working on producing figures to complement the text. We will update this document as soon as the figures are ready. Note also that this is a provisional document and some final corrections may be made for accuracy and/or completeness. Feedback is most welcome!</em></p>

<hr></hr><h3>Overview</h3>

<p>The previous step produced a list of ActiveRegions that showed some evidence of possible variation (see <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=4147">step 1 documentation</a>). Now, we need to process each Active Region in order to generate a list of possible haplotypes based on the sequence data we have for that region.</p>

<p>To do so, the program first builds an assembly graph for each active region (determined in the previous step) using the reference sequence as a template. Then, it takes each read in turn and attempts to match it to a segment of the graph. Whenever portions of a read do not match the local graph, the program adds new nodes to the graph to account for the mismatches. After this process has been repeated with many reads, it typically yields a complex graph with many possible paths. However, because the program keeps track of how many reads support each path segment, we can select only the most likely (well-supported) paths. These likely paths are then used to build the haplotype sequences which will be used to call variants and assign per-sample genotypes in the next steps.</p>

<hr></hr><h3>1. Reference graph assembly</h3>

<p>First, we construct the reference assembly graph, which starts out as a simple <strong>directed DeBruijn graph</strong>. This involves decomposing the reference sequence into a succession of <strong>kmers</strong> (pronounced "kay-mers"), which are small sequence subunits that are <strong>k</strong> bases long. Each kmer sequence overlaps the previous kmer by k-1 bases. The resulting graph can be represented as a series of nodes and connecting edges indicating the sequential relationship between the adjacent bases. At this point, all the connecting edges have a weight of 0.</p>

<p>In addition to the graph, we also build a hash table of unique kmers, which we use to keep track of the position of nodes in the graph. At the beginning, the hash table only contains unique kmers found in the reference sequence, but we will add to it in the next step.</p>

<p><strong>A note about kmer size:</strong> by default, the program will attempt to build two separate graphs, using kmers of 10 and 25 bases in size, respectively, but other kmer sizes can be specified from the command line with the <code class="code codeInline" spellcheck="false">-kmerSize</code> argument. The final set of haplotypes will be selected from the union of the graphs obtained using each k.</p>

<hr></hr><h3>2. Threading reads through the graph</h3>

<p>This is where our simple reference graph turns into a <strong>read-threading graph</strong>, so-called because we're going to take each read in turn and try to match it to a path in the graph.</p>

<p>We start with the first read and compare its first kmer to the hash table to find if it has a match. If there is a match, we look up its position in the reference graph and record that position. If there is no match, we consider that it is a new unique kmer, so we add that unique kmer to the hash table and add a new node to the graph. In both cases, we then move on and repeat the process with the next kmer in the read until we reach the end of the read.</p>

<p>When two consecutive kmers in a read belong to two nodes that were already connected by an edge in the graph, we increase the weight of that edge by 1. If the two nodes were not connected yet, we add a new edge to the graph with a starting weight of 1. As we repeat the process on each read in turn, edge weights will accumulate along the paths that are best supported by the read data, which will help us select the most likely paths later on.</p>

<p><strong>Note on graph complexity, cycles and non-unique kmers</strong></p>

<p>For this process to work properly, we need the graph to be sufficiently complex (where the number of non-unique k-mers is less that 4-fold the number of unique kmers found in the data) and without cycles. In certain genomic regions where there are a lot of repeated sequences, these conditions may not be met, because repeats cause cycles and diminish the number of available unique kmers. If none of the kmer sizes provided results in a viable graph (complex enough and without cycles) the program will automatically try the operation again with larger kmer sizes. Specifically, we take the largest k provided by the user (or by the default settings) and increase it by 10 bases. If no viable graph can be obtained after iterating over increased kmer sizes 6 times, we give up and skip the active region entirely.</p>

<hr></hr><h3>3. Graph refinement</h3>

<p>Once all the reads have been threaded through the graph, we need to clean it up a little. The main cleaning-up operation is called <strong>pruning</strong> (like the gardening technique). The goal of the pruning operation is to remove noise due to errors. The basic idea is that sections of the graph that are supported by very few reads are most probably the result of stochastic errors, so we are going to remove any sections that are supported by fewer than a certain threshold number of reads. By default the threshold value is 2, but this can be controlled from the command line using the <a rel="nofollow" href="http://www.broadinstitute.org/gatk/gatkdocs/org_broadinstitute_sting_gatk_walkers_haplotypecaller_HaplotypeCaller.html#-minPruning"><code class="code codeInline" spellcheck="false">-minPruning</code></a> argument. In practice, this means that linear chains in the graph (linear sequence of vertices and edges without any branching) where all edges have fewer than 2 supporting reads will be removed. Increasing the threshold value will lead to faster processing and higher specificity, but will decrease sensitivity. Decreasing this value will do the opposite, decreasing specificity but increasing sensitivity.</p>

<p>At this stage, the program also performs graph refinement operations, such as recovering <strong>dangling heads and tails</strong> from the splice junctions to compensate for issues that are related to limitations in graph assembly.</p>

<p>Note that if you are calling multiple samples together, the program also looks at how many of the samples support each segment, and only prunes segments for which fewer than a certain number of samples have the minimum required number of supporting reads. By default this sample number is 1, so as long as one sample in the cohort passes the pruning threshold, the segment will NOT be pruned. This is designed to avoid losing singletons (variants that are unique to a single sample in a cohort). This parameter can also be controlled from the command line using the <code class="code codeInline" spellcheck="false">-minPruningSamples</code> argument, but keep in mind that increasing the default value may lead to decreased sensitivity.</p>

<hr></hr><h3>4. Select best haplotypes</h3>

<p>Now that the graph is all cleaned up, the program builds haplotype sequences by traversing all possible paths in the graph and calculates a likelihood score for each one. This score is calculated as the product of transition probabilities of the path edges, where the transition probability of an edge is computed as the number of reads supporting that edge divided by the sum of the support of all edges that share that same source vertex.</p>

<p>In order to limit the amount of computation needed for the next step, we limit the number of haplotypes that will be considered for each value of k (remember that the program builds graphs for multiple kmer sizes). This is easy to do since we conveniently have scores for each haplotype; all we need to do is select the N haplotypes with the best scores. By default that number is very generously set to 128 (so the program would proceed to the next step with up to 128 haplotypes per value of k) but this can be adjusted from the command line using the <code class="code codeInline" spellcheck="false">-maxNumHaplotypesInPopulation</code> argument. You would mainly want to decrease this number in order to improve speed; increasing that number would rarely be reasonable, if ever.</p>

<hr></hr><h3>5. Identify potential variation sites</h3>

<p>Once we have a list of plausible haplotypes, we perform a Smith-Waterman alignment (SWA) of each haplotype to the original reference sequence across the active region in order to reconstruct a CIGAR string for the haplotype. Note that indels will be left-aligned; that is, their start position will be set as the leftmost position possible.</p>

<p>This finally yields the potential variation sites that will be put through the variant modeling step next, bringing us back to the "classic" variant calling methods (as used by GATK's UnifiedGenotyper and Samtools' mpileup). Note that this list of candidate sites is essentially a super-set of what will eventually be the final set of called variants. Every site that will be called variant is in the super-set, but not every site that is in the super-set will be called variant.</p>
