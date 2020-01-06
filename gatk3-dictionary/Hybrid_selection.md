## Hybrid selection

By Geraldine_VdAuwera

<p>Hybrid selection is a method that enables selection of specific sequences from a pool of genomic DNA for targeted sequencing analyses via pull-down assays.  Typical applications include the selection of exome sequences or pathogen-specific sequences in complex biological samples. Hybrid selection involve the use <strong>baits</strong> to select desired fragments.</p>

<p>Briefly, baits are RNA (or sometimes DNA) molecules synthesized with biotinylated nucleotides. The biotinylated nucleotides are ligands for streptavidin enabling enabling RNA:DNA hybrids to be captured in solution. The hybridization targets are sheared genomic DNA fragments, which have been "polished" with synthetic adapters to facilitate PCR cloning downstream. Hybridization of the baits with the denatured targets is followed by selective capture of the RNA:DNA "hybrids" using streptavidin-coated beads via pull-down assays or columns.</p>

<p>Systematic errors, ultimately leading to sequence bias and incorrect variant calls, can arise at several steps. See the GATK dictionary entries <a rel="nofollow" href="http://gatkforums.broadinstitute.org/gatk/discussion/6333">bait bias</a> and <a rel="nofollow" href="http://gatkforums.broadinstitute.org/gatk/discussion/6332">pre-adapter artifacts</a> for more details.</p>

<p>Please see the following <a rel="nofollow" href="http://www.nature.com/nbt/journal/v27/n2/abs/nbt.1523.html">reference</a> for the theory behind this technique.</p>
