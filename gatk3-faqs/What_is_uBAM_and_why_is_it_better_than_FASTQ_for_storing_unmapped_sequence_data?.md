## What is uBAM and why is it better than FASTQ for storing unmapped sequence data?

By Geraldine_VdAuwera

<p>Most sequencing providers generate FASTQ files with the raw unmapped read sequences, so that is the most common form in which the data is input into the mapping step of the pre-processing pipeline. This is not ideal because among other flaws, much of the metadata associated with sequencing runs cannot be stored in FASTQ files, unlike BAM files which can store more information. See <a rel="nofollow" href="http://blastedbio.blogspot.co.uk/2011/10/fastq-must-die-long-live-sambam.html">this blog post</a> for an overview of the many problems associated with the FASTQ format.</p>

<p>At the Broad Institute, we generate unmapped BAM (uBAM) files directly from the Illumina basecalls in order to keep all metadata in one place, and we do not write the data to FASTQ files at any point. This involves a slightly more complex workflow than is shown in the general Best Practices diagram. See <a rel="nofollow" href="https://www.broadinstitute.org/gatk/events/slides/1506/GATKwr8-A-3-GATK_Best_Practices_and_Broad_pipelines.pdf">this presentation</a> for more details of how this works.</p>

<p>In case you're wondering, we still show the FASTQ-based workflow as the default in most of our documentation  because it is by far the most commonly-used workflow, and we want to keep the documentation accessible for our more novice users.</p>
