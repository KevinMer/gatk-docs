## Read groups

By Geraldine_VdAuwera

<p>There is no formal definition of what is a read group, but in practice, this term refers to a set of reads that were generated from a single run of a sequencing instrument.</p>

<p>In the simple case where a single library preparation derived from a single biological sample was run on a single lane of a flowcell, all the reads from that lane run belong to the same read group. When multiplexing is involved, then each subset of reads originating from a separate library run on that lane will constitute a separate read group.</p>

<p>Read groups are identified in the SAM/BAM /CRAM file by a number of tags that are defined in the <a rel="nofollow" href="http://samtools.github.io/hts-specs/">official SAM specification</a>. These tags, when assigned appropriately, allow us to differentiate not only samples, but also various technical features that are associated with artifacts. With this information in hand, we can mitigate the effects of those artifacts during the duplicate marking and base recalibration steps. The GATK requires several read group fields to be present in input files and will fail with errors if this requirement is not satisfied. See <a rel="nofollow" href="http://www.broadinstitute.org/gatk/guide/article?id=59">this article</a> for common problems related to read groups.</p>

<p>To see the read group information for a BAM file, use the following command.</p>

<pre class="code codeBlock" spellcheck="false">samtools view -H sample.bam | grep '@RG'
</pre>

<p>This prints the lines starting with <code class="code codeInline" spellcheck="false">@RG</code> within the header, e.g. as shown in the example below.</p>

<pre class="code codeBlock" spellcheck="false">@RG ID:H0164.2  PL:illumina PU:H0164ALXX140820.2    LB:Solexa-272222    PI:0    DT:2014-08-20T00:00:00-0400 SM:NA12878  CN:BI
</pre>

<hr></hr><h3>Meaning of the read group fields required by GATK</h3>

<ul><li><p><code class="code codeInline" spellcheck="false">ID</code> = <strong>Read group identifier</strong><br>
This tag identifies which read group each read belongs to, so each read group's <code class="code codeInline" spellcheck="false">ID</code> must be unique. It is referenced both in the read group definition line in the file header (starting with <code class="code codeInline" spellcheck="false">@RG</code>) and in the <code class="code codeInline" spellcheck="false">RG:Z</code> tag for each read record. Note that some Picard tools have the ability to modify <code class="code codeInline" spellcheck="false">ID</code>s when merging SAM files in order to avoid collisions. In Illumina data, read group <code class="code codeInline" spellcheck="false">ID</code>s are composed using the flowcell + lane name and number, making them a globally unique identifier across all sequencing data in the world.<br><em>Use for BQSR:</em> <code class="code codeInline" spellcheck="false">ID</code> is the lowest denominator that differentiates factors contributing to technical batch effects: therefore, a read group is effectively treated as a separate run of the instrument in data processing steps such as base quality score recalibration, since they are assumed to share the same error model.</p></li>
<li><p><code class="code codeInline" spellcheck="false">PU</code> = <strong>Platform Unit</strong> <br>
The <code class="code codeInline" spellcheck="false">PU</code> holds three types of information, the {FLOWCELL_BARCODE}.{LANE}.{SAMPLE_BARCODE}.  The {FLOWCELL_BARCODE} refers to the unique identifier for a particular flow cell.  The {LANE} indicates the lane of the flow cell and the {SAMPLE_BARCODE} is a sample/library-specific identifier.  Although the <code class="code codeInline" spellcheck="false">PU</code> is not required by GATK but takes precedence over <code class="code codeInline" spellcheck="false">ID</code> for base recalibration if it is present. In the example shown earlier, two read group fields, <code class="code codeInline" spellcheck="false">ID</code> and <code class="code codeInline" spellcheck="false">PU</code>, appropriately differentiate flow cell lane, marked by <code class="code codeInline" spellcheck="false">.2</code>, a factor that contributes to batch effects.</p></li>
<li><p><code class="code codeInline" spellcheck="false">SM</code> = <strong>Sample</strong><br>
 The name of the sample sequenced in this read group. GATK tools treat all read groups with the same <code class="code codeInline" spellcheck="false">SM</code> value as containing sequencing data for the same sample, and this is also the name that will be used for the sample column in the VCF file. Therefore it's critical that the <code class="code codeInline" spellcheck="false">SM</code> field be specified correctly. When sequencing pools of samples, use a pool name instead of an individual sample name. Note, when we say pools, we mean samples that are not individually barcoded. In the case of multiplexing (often confused with pooling) where you know which reads come from each sample and you have simply run the samples together in one lane, you can keep the SM tag as the sample name and not the "pooled name".</p></li>
<li><p><code class="code codeInline" spellcheck="false">PL</code> = <strong>Platform/technology used to produce the read</strong> <br>
This constitutes the only way to know what sequencing technology was used to generate the sequencing data. Valid values: ILLUMINA, SOLID, LS454, HELICOS and PACBIO.</p></li>
<li><p><code class="code codeInline" spellcheck="false">LB</code> = <strong>DNA preparation library identifier</strong> <br>
MarkDuplicates uses the LB field to determine which read groups might contain molecular duplicates, in case the same DNA library was sequenced on multiple lanes.</p></li>
</ul><p>If your sample collection's BAM files lack required fields or do not differentiate pertinent factors within the fields, use Picard's <a rel="nofollow" href="http://broadinstitute.github.io/picard/command-line-overview.html#AddOrReplaceReadGroups">AddOrReplaceReadGroups</a> to add or appropriately rename the read group fields as outlined <a rel="nofollow" href="http://gatkforums.broadinstitute.org/discussion/2909/">here</a>.</p>

<hr></hr><h3>Deriving <code class="code codeInline" spellcheck="false">ID</code> and <code class="code codeInline" spellcheck="false">PU</code> fields from read names</h3>

<p>Here we illustrate how to derive both <code class="code codeInline" spellcheck="false">ID</code> and <code class="code codeInline" spellcheck="false">PU</code> fields from read names as they are formed in the data produced by the Broad Genomic Services pipelines (other sequence providers may use different naming conventions). We break down the common portion of two different read names from a sample file. The unique portion of the read names that come after flow cell lane, and separated by colons, are tile number, x-coordinate of cluster and y-coordinate of cluster.</p>

<pre class="code codeBlock" spellcheck="false">H0164ALXX140820:2:1101:10003:23460
H0164ALXX140820:2:1101:15118:25288
</pre>

<p>Breaking down the common portion of the query names:</p>

<pre class="code codeBlock" spellcheck="false">H0164____________ #portion of @RG ID and PU fields indicating Illumina flow cell
_____ALXX140820__ #portion of @RG PU field indicating barcode or index in a multiplexed run
_______________:2 #portion of @RG ID and PU fields indicating flow cell lane
</pre>

<hr></hr><h3>Multi-sample and multiplexed example</h3>

<p>Suppose I have a trio of samples: MOM, DAD, and KID.  Each has two DNA libraries prepared, one with 400 bp inserts and another with 200 bp inserts.  Each of these libraries is run on two lanes of an Illumina HiSeq, requiring 3 x 2 x 2 = 12 lanes of data.  When the data come off the sequencer, I would create 12 bam files, with the following <code class="code codeInline" spellcheck="false">@RG</code> fields in the header:</p>

<pre class="code codeBlock" spellcheck="false">Dad's data:
@RG     ID:FLOWCELL1.LANE1      PL:ILLUMINA     LB:LIB-DAD-1 SM:DAD      PI:200
@RG     ID:FLOWCELL1.LANE2      PL:ILLUMINA     LB:LIB-DAD-1 SM:DAD      PI:200
@RG     ID:FLOWCELL1.LANE3      PL:ILLUMINA     LB:LIB-DAD-2 SM:DAD      PI:400
@RG     ID:FLOWCELL1.LANE4      PL:ILLUMINA     LB:LIB-DAD-2 SM:DAD      PI:400

Mom's data:
@RG     ID:FLOWCELL1.LANE5      PL:ILLUMINA     LB:LIB-MOM-1 SM:MOM      PI:200
@RG     ID:FLOWCELL1.LANE6      PL:ILLUMINA     LB:LIB-MOM-1 SM:MOM      PI:200
@RG     ID:FLOWCELL1.LANE7      PL:ILLUMINA     LB:LIB-MOM-2 SM:MOM      PI:400
@RG     ID:FLOWCELL1.LANE8      PL:ILLUMINA     LB:LIB-MOM-2 SM:MOM      PI:400

Kid's data:
@RG     ID:FLOWCELL2.LANE1      PL:ILLUMINA     LB:LIB-KID-1 SM:KID      PI:200
@RG     ID:FLOWCELL2.LANE2      PL:ILLUMINA     LB:LIB-KID-1 SM:KID      PI:200
@RG     ID:FLOWCELL2.LANE3      PL:ILLUMINA     LB:LIB-KID-2 SM:KID      PI:400
@RG     ID:FLOWCELL2.LANE4      PL:ILLUMINA     LB:LIB-KID-2 SM:KID      PI:400
</pre>

<p>Note the hierarchical relationship between read groups (unique for each lane) to libraries (sequenced on two lanes) and samples (across four lanes, two lanes for each library).</p>
