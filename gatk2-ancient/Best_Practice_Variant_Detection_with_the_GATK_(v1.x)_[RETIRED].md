## Best Practice Variant Detection with the GATK (v1.x) [RETIRED]

By ebanks

<h2>Data Processing Pipeline Script</h2>
<p>The Data Processing Pipeline is a Queue script that performs all raw data processing described in this page following our most current stable recommendations for best practices.<br></p>
<h2>Introduction</h2>
<p>Our current best practice for making SNP and indel calls is divided into 4 sequential steps: initial mapping, refinement of the initial reads, multi-sample indel and SNP calling, and finally variant quality score recalibration.  These steps are the same for targeted resequencing, whole exomes, deep whole genomes, and low-pass whole genomes.  The exact commands for each tool are available on the individual tool's wiki entry.  <a rel="nofollow" href="http://gsa-stage:8080/about#typical-workflows">See here for a visual representation of the workflow.</a><br></p>
<p><b>Note that, due to the specific attributes of a project, that the the specific values used in each of the commands may need to be selected by the analyst.  Care should be taken by the analyst running our tools to understand what each parameter does and to evaluate which value best fits his/her data.</b><br></p>
<h3>Lanes, Samples, Cohort</h3>
<p>There are four major organizational units for next-generation DNA sequencing processes:<br></p>
<dl><dt> Lane </dt><dd> The basic machine unit for sequencing.  The lane reflects the basic independent run of an NGS machine.  For Illumina machines, this is the physical sequencing lane.  <br></dd></dl><dl><dt> Library </dt><dd> A unit of DNA preparation that at some point is physically pooled together.  Multiple lanes can be run from aliquots from the same library.  The DNA library and its preparation is the natural unit that is being sequenced.  For example, if the library has limited complexity, then many sequences are duplicated and will result in a high duplication rate across lanes. <br></dd></dl><dl><dt> Sample </dt><dd> A single individual, such as human CEPH NA12878.  Multiple libraries with different properties can be constructed from the original sample DNA source.  Here we treat samples as independent individuals whose genome sequence we are attempting to determine.  From this perspective, tumor / normal samples are different despite coming from the same individual.<br></dd></dl><dl><dt> Cohort </dt><dd> A collection of samples being analyzed together.  This organizational unit is the most subjective and depends intimately on the design goals of the sequencing project.  For population discovery projects like the 1000 Genomes, the analysis cohort is the ~100 individual in each population.  For exome projects with many samples (e.g., ESP with 800 EOMI samples) deeply sequenced we divide up the complete set of samples into cohorts of ~50 individuals for multi-sample analyses.  <br></dd></dl><p>This document describes how to call variation within a single analysis cohort, comprised for one or many samples, each of one or many libraries that were sequenced on at least one lane of an NGS machine. <br></p><p>Note that many GATK commands can be run at the lane level, but will give better results seeing all of the data for a single sample, or even all of the data for all samples.  Unfortunately, there's a trade-off in computational cost by running these commands across all of your data simultaneously.<br></p>
<h3>Testing data: 64x HiSeq on chr20 for NA12878</h3>
<p>In order to help individuals get up to speed, evaluate their command lines, and generally become familiar with the GATK tools we recommend you download the raw and realigned, recalibrated <a rel="nofollow" href="/gsa/wiki/index.php/NA12878_test_data" title="NA12878 test data">NA12878 test data</a>.  It should be possible to apply all of the approaches outlined below to get excellent results for realignment, recalibration, SNP calling, indel calling, filtering, and variant quality score recalibration using this data.<br></p>
<h2>Phase I: Raw data processing</h2>
<h3>Initial read mapping</h3>
<p>The GATK data processing pipeline assumes that one of the many NGS read aligners (see <a rel="nofollow" href="http://bib.oxfordjournals.org/content/11/5/473.abstract">[1]</a> for a review) has been applied to your raw FASTQ files.  For Illumina data we recommend <a rel="nofollow" href="http://bio-bwa.sourceforge.net/">BWA</a> because it is accurate, fast, well-supported, open-source, and emits BAM files natively.<br></p><p><br><br></p>
<h3>Raw BAM to realigned, recalibrated BAM</h3>
<p>The three key tools here are <a rel="nofollow" href="http://gsa-stage:8080/gatkdocs/org_broadinstitute_sting_gatk_walkers_bqsr_BaseQualityScoreRecalibrator.html" title="Base quality score recalibration">Base quality score recalibration</a>, <a rel="nofollow" href="http://gsa-stage:8080/gatkdocs/org_broadinstitute_sting_gatk_walkers_indels_IndelRealigner.html" title="Local realignment around indels">Local realignment around indels</a>, and <a rel="nofollow" href="http://picard.sourceforge.net/command-line-overview.shtml#MarkDuplicates%7CPicard's">MarkDuplicates</a>.  Although ideally one follows the recommended workflow, in practice MarkDuplicates can be run before local realignment, in order to handle cases where duplicates overlapping indels get marginally different alignments (unlikely but possible) and so will not be considered as potential duplicates because MarkDuplicates looks only at read pair start and stop positions.  There are several options here, from the easy and fast basic protocol to the more.<br></p><p>There are two types of realignment:<br></p>
<ul><li> Realignment only at known sites, which is very efficient, can operate with little coverage (1x per lane genome wide) but can only realign reads at known indels.<br></li><li> Fully local realignment uses mismatching bases to determine if a site should be realigned, and relies on sufficient coverage to discover the correct indel allele in the reads for alignment.  It is much slower (involves SW step) but can discover new indel sites in the reads.  If you have a database of known indels (for human, this database is extensive) then at this stage you would also include these indels during realignment, which vastly improves sensitivity, specificity, and speed.<br></li></ul><h4>Previous recommendation: lane-level recalibration, sample-level realignment</h4>
<p>This is the protocol we used at the Broad Institute for the last year.<br></p>
<pre class="code codeBlock" spellcheck="false"><br>for each lane.bam<br>    dedup.bam &lt;- MarkDuplicate(lane.bam)<br>    recal.bam &lt;- recal(dedup.bam)<br><br>for each sample<br>    lanes.bam &lt;- merged recal.bam's for sample<br>    dedup.bam &lt;- MarkDuplicates(lanes.bam)<br>    realigned.bam &lt;- realign(dedup.bam) [with known sites if possible]<br></pre>
<h4>Fast: lane-level realignment at known sites only and lane-level recalibration</h4>
<p>This protocol adds lane-level local realignment around known indels, making it very fast (no sample level processing) and gives better results for human samples than the previous recommendation:<br></p>
<pre class="code codeBlock" spellcheck="false"><br>for each lane.bam<br>    realigned.bam &lt;- realign(lane.bam) [at only known sites]<br>    dedup.bam &lt;- MarkDuplicate(realigned.bam)<br>    recal.bam &lt;- recal(dedup.bam)<br><br>for each sample<br>    recals.bam &lt;- merged lane-level recal.bam's for sample<br>    dedup.bam &lt;- MarkDuplicates(recals.bam)<br></pre>
<h4>Fast + sample-level realignment</h4>
<p>This protocol adds sample-level realignment after library / sample level dedupping, so that novel indels in each sample can be discovered and realigned around.<br></p>
<pre class="code codeBlock" spellcheck="false"><br>for each lane.bam<br>    realigned.bam &lt;- realign(lane.bam) [at only known sites]<br>    dedup.bam &lt;- MarkDuplicate(realigned.bam)<br>    recal.bam &lt;- recal(dedup.bam)<br><br>for each sample<br>    recals.bam &lt;- merged lane-level recal.bam's for sample<br>    dedup.bam &lt;- MarkDuplicates(recals.bam)<br>    realigned.bam &lt;- realign(dedup.bam) [with known sites included if available]<br></pre>
<h4>Better: sample-level realignment with known indels and recalibration</h4>
<p>Rather than doing the lane level cleaning and recalibration, this process aggregates all of the reads for each sample and then does a full dedupping, realign, and recalibration, yielding the best single-sample results.  The big change here is sample-level cleaning followed by recalibration, giving you the most accurate quality scores possible for a single sample. <br></p>
<pre class="code codeBlock" spellcheck="false"><br>for each sample<br>    lanes.bam &lt;- merged lane.bam's for sample<br>    dedup.bam &lt;- MarkDuplicates(lanes.bam)<br>    realigned.bam &lt;- realign(dedup.bam) [with known sites included if available]<br>    recal.bam &lt;- recal(realigned.bam)<br></pre>
<h4>Best: multi-sample realignment with known sites and recalibration</h4>
<p>Finally, if you really want to get the absolute best results, whatever the computational cost, then we recommend doing multiple sample realignment so that novel indels in one sample help to realign reads in other samples.  Although not generally necessary for deep sequencing data, this is important for low-coverage multi-sample SNP calling projects like the 1000 Genomes Project.  Note that the computational cost here is so extreme that we only do this analysis in special circumstances, such as large-scale data freeze for the project.  <br></p><p>Note that for contrastive calling projects -- such as cancer tumor/normals -- that we recommend cleaning both the tumor and the normal together in general to avoid slight alignment differences between the two tissue types.<br></p>
<pre class="code codeBlock" spellcheck="false"><br>for each sample<br>    lanes.bam &lt;- merged lane.bam's for sample<br>    dedup.bam &lt;- MarkDuplicates(lanes.bam)<br><br>samples.bam &lt;- merged dedup.bam's for all samples<br>realigned.bam &lt;- realign(samples.bam)<br>recal.bam &lt;- recal(realigned.bam)<br></pre>
<h4>Misc. notes on the process</h4>
<ul><li> MarkDuplicates needs only be run at the library level. So the sample-level dedupping isn't necessary if you only ever a library on a single lane.  If you run the sample library on many lanes (as can be necessary for whole exome, for example), you should dedup at the library level.<br></li><li> The base quality score recalibrator is read group aware, so running it on a merged BAM files containing multiple read groups is the same as running it on each bam file individually.  There's some memory cost (so it's best not to recalibrate 10000 RGs simultaneously) but for reasonable projects this is a fine.<br></li><li> Local realignment preserves read meta-data, so you can realign and then recalibrate just fine.<br></li></ul><h2>Initial variant discovery and genotyping</h2>
<h3>Input BAMs for variant discovery and genotyping</h3>
<p>After the raw data processing step, the GATK variant detection process assumes that you have aligned, duplicate marked, and recalibrated BAM files for all of the samples in your cohort.  Because the GATK can dynamically merge BAM files, it isn't critical to have merged files by lane into sample bams, or even samples bams into cohort bams.  In general we try to create sample level bams for deep data sets (deep WG or exomes) and merged cohort files by chromosome for WG low-pass.  A nice size for BAMs is 10-300 Gb, just for organizing on disk.  <br></p><p>For this part of the this document, I'm going to assume that you have a single realigned, recalibrated, dedupped BAM per sample, called sampleX.bam, for X from 1 to N samples in your cohort.  Note that some of the data processing steps, such as multiple sample local realignment, will merge BAMS for many samples into a single BAM.  If you've gone down this route, you just need to modify the GATK commands as necessary to take not multiple BAMs, one for each sample, but a single BAM for all samples.<br></p>
<h3>Multi-sample SNP and indel calling</h3>
<p>The next step in the standard GATK data processing pipeline, whole genome or targeted, deep or shallow, is to apply the <a rel="nofollow" href="http://gsa-stage:8080/gatkdocs/org_broadinstitute_sting_gatk_walkers_genotyper_UnifiedGenotyper.html" title="Unified genotyper">Unified genotyper</a> to identify sites among the cohort samples that are statistically non-reference.  This will produce a multi-sample <a rel="nofollow" href="http://www.1000genomes.org/wiki/Analysis/variant-call-format" title="VCF Format">VCF file</a>, with sites discovered across samples and genotypes assigned to each sample in the cohort.  It's in this stage that we use the meta-data in the BAM files extensively -- read groups for reads, with samples, platforms, etc -- to enable us to do the multi-sample merging and genotyping correctly.  It was a pain for data processing, yes, but now life is easy for downstream calling and analysis.<br></p><p>Note that by default the Unified Genotyper calls SNPs only.  To enable the indel calling capabilities (in addition to SNPs) instead use the -glm BOTH argument.<br></p>
<h4>Selecting an appropriate quality score threshold</h4>
<p>A common question is the confidence score threshold to use for variant detection.  We recommend:<br></p>
<dl><dt> Deep (&gt; 10x coverage per sample) data </dt><dd> we recommend a minimum confidence score threshold of Q30.<br></dd><dt> Shallow (&lt; 10x coverage per sample) data </dt><dd> because variants have by necessity lower quality with shallower coverage, we recommend a min. confidence score of Q4 in project with 100 samples or fewer and Q10 otherwise.<br></dd></dl><h3>Protocol</h3>
<pre class="code codeBlock" spellcheck="false"><br>raw.vcf &lt;- unifiedGenotyper(sample1.bam, sample2.bam, ..., sampleN.bam)<br></pre>
<h2>Integrating analyses: getting the best call set possible</h2>
<p>This raw VCF file should be as sensitive to variation as you'll get without imputation.  At this stage, you can assess things like sensitivity to known variant sites or genotype chip concordance.  The problem is that the raw VCF will have many sites that aren't really genetic variants but are machine artifacts that make the site statistically non-reference.  All of the subsequent steps are designed to separate out the FP machine artifacts from the TP genetic variants.<br></p>
<h3>Whole Genome Shotgun experiments</h3>
<p>The tool used here is the <a rel="nofollow" href="http://gsa-stage:8080/gatkdocs/org_broadinstitute_sting_gatk_walkers_variantrecalibration_VariantRecalibrator.html" title="Variant quality score recalibration">Variant quality score recalibration</a> which builds an adaptive error model using known variant sites and then apply this model to estimate the probability that each variant is a true genetic variant or a machine artifact. The tool builds a separate model for SNPs and indels and must be run twice in succession in order to recalibrate both classes of variants. One major improvement from previous recommended protocols is that hand filters do not need to be applied at any point in the process now. All filtering criteria are learned from the data itself. <br></p>
<h5>Analysis ready VCF protocol</h5>
<pre class="code codeBlock" spellcheck="false"><br>snp.model &lt;- BuildErrorModelWithVQSR(raw.vcf, SNP)<br>indel.model &lt;- BuildErrorModelWithVQSR(raw.vcf, INDEL)<br>recalibratedSNPs.rawIndels.vcf &lt;- ApplyRecalibration(raw.vcf, snp.model, SNP)<br>analysisReady.vcf &lt;- ApplyRecalibration(recalibratedSNPs.rawIndels.vcf, indel.model, INDEL)<br></pre>
<p><b>Common VariantRecalibrator command</b><br></p><p>Please review the <a rel="nofollow" href="http://gsa-stage:8080/gatkdocs/org_broadinstitute_sting_gatk_walkers_variantrecalibration_VariantRecalibrator.html" title="Variant quality score recalibration">Variant quality score recalibration</a> wiki page for details of how one specifies truth and training sets. <br></p>
<pre class="code codeBlock" spellcheck="false"><br>java -Xmx4g -jar GenomeAnalysisTK.jar \<br>   -T VariantRecalibrator \<br>   -R path/to/reference/human_g1k_v37.fasta \<br>   -input,VCF raw.vcf \<br>   [SPECIFY TRUTH AND TRAINING SETS]<br>   -recalFile path/to/output.recal \<br>   -tranchesFile path/to/output.tranches \<br>   -rscriptFile path/to/output.plots.R<br>   [SPECIFY WHICH ANNOTATIONS TO USE IN MODELING]<br></pre>
<h5>SNP specific recommendations</h5>
<p>For SNPs we use both HapMap v3.3 and the Omni chip array from the 1000 Genomes Project as training data. These datasets are available in the <a rel="nofollow" href="/gsa/wiki/index.php/GATK_resource_bundle" title="GATK resource bundle">GATK resource bundle</a>.<br></p><p>Arguments for VariantRecalibrator command:<br></p>
<pre class="code codeBlock" spellcheck="false"><br>   -resource:hapmap,VCF,known=false,training=true,truth=true,prior=15.0 hapmap_3.3.b37.sites.vcf \<br>   -resource:omni,VCF,known=false,training=true,truth=false,prior=12.0 1000G_omni2.5.b37.sites.vcf \<br>   -resource:dbsnp,VCF,known=true,training=false,truth=false,prior=6.0 dbsnp_135.b37.vcf \<br>   -an QD -an HaplotypeScore -an MQRankSum -an ReadPosRankSum -an FS -an MQ -an InbreedingCoeff -an DP \<br>   -mode SNP \<br></pre>
<p>Note that, for the above to work, the input vcf needs to be annotated with the corresponding values (QD, FS, MQ, etc.). If any of these values are somehow missing, then VariantAnnotator needs to be run first so that VariantRecalibration can run properly.<br></p><p>Also, note that some of these annotations might not be the best for your particular dataset. For example, InbreedingCoeff is a population level statistic that requires at least 10 samples in order to be calculated.<br></p><p>Using the provided sites-only truth data files is important here as parsing the genotypes for VCF files with many samples increases the runtime of the tool significantly.<br></p>
<h5>Indel specific recommendations</h5>
<p>When modeling indels with the VQSR we use a training dataset that was created at the Broad by strictly curating the (Mills, Devine, Genome Research, 2011) dataset as as well as adding in very high confidence indels from the 1000 Genomes Project. This dataset is available in the <a rel="nofollow" href="/gsa/wiki/index.php/GATK_resource_bundle" title="GATK resource bundle">GATK resource bundle</a>.<br></p><p>Arguments for VariantReacalibrator:<br></p>
<pre class="code codeBlock" spellcheck="false"><br>   -resource:mills,VCF,known=true,training=true,truth=true,prior=12.0 Mills_and_1000G_gold_standard.indels.b37.sites.vcf \<br>   -an QD -an FS -an HaplotypeScore -an ReadPosRankSum -an InbreedingCoeff \<br>   -mode INDEL \<br></pre>
<p>Note that indels use a different set of annotations than SNPs. The annotations related to mapping quality have been removed since there is a conflation with the length of an indel in a read and the degradation in mapping quality that is assigned to the read by the aligner.<br></p><p><b>Common ApplyRecalibration command</b><br></p><p>The <a rel="nofollow" href="http://gsa-stage:8080/gatkdocs/org_broadinstitute_sting_gatk_walkers_variantrecalibration_VariantRecalibrator.html" title="Variant quality score recalibration">Variant quality score recalibration</a> wiki page has an example command for the ApplyRecalibration step which works for both SNPs and indels. One would just need to add -mode SNP (the default) or -mode INDEL to the command line.<br></p>
<h3>Whole Exome experiments</h3>
<p>For exome SNPs, the tool used here, as in the whole genome case, is the <a rel="nofollow" href="/gsa/wiki/index.php/Variant_quality_score_recalibration" title="Variant quality score recalibration">Variant quality score recalibration</a> which builds an adaptive error model using known variant sites and then apply this model to estimate the probability that each variant is a true genetic variant or a machine artifact. In our testing we've found that in order to achieve the best exome results one needs to use an exome SNP callset with at least 30 samples. For users with experiments containing fewer exome samples there are several options to explore:<br></p>
<ul><li> Add additional samples for variant calling, either by sequencing additional samples or using publicly available exome bams from the 1000 Genomes Project (this option is used by the Broad exome production pipeline)<br></li><li> Use the VQSR with the smaller callset but experiment with the precise argument settings (try adding --maxGaussians 4 --percentBad 0.05 to your command line, for example)<br></li><li> Use hard filters (detailed below).<br></li></ul><p>For exome indels there generally isn't enough data to build a robust error model and so we recommend applying hand filters to the exome indels. The protocol is to first use <a rel="nofollow" href="/gsa/wiki/index.php/SelectVariants" title="SelectVariants">SelectVariants</a> to separate out SNPs and indels. Then recalibrate the SNPs and hand filter the indels in parallel. Finally, use <a rel="nofollow" href="/gsa/wiki/index.php/CombineVariants" title="CombineVariants">CombineVariants</a> to combine the high quality variants back into one VCF file.   <br></p>
<h5>Analysis ready VCF protocol</h5>
<pre class="code codeBlock" spellcheck="false"><br>raw.snps.vcf &lt;- Select(raw.vcf, SNP)<br>raw.indels.vcf &lt;- Select(raw.vcf, INDEL)<br>snp.model &lt;- BuildErrorModelWithVQSR(raw.snps.vcf)<br>recalibratedSNPs.vcf &lt;- ApplyRecalibration(raw.snps.vcf, snp.model)<br>filteredIndels.vcf &lt;- Filter(raw.indels.vcf)<br>analysisReady.vcf &lt;- CombineTogether(recalibratedSNPs.vcf, filteredIndels.vcf)<br></pre>
<h5>SNP specific recommendations</h5>
<p>For SNPs we use both HapMap v3.3 and the Omni chip array from the 1000 Genomes Project as training data. These datasets are available in the <a rel="nofollow" href="/gsa/wiki/index.php/GATK_resource_bundle" title="GATK resource bundle">GATK resource bundle</a>.<br></p><p>Arguments for VariantRecalibrator command:<br></p>
<pre class="code codeBlock" spellcheck="false"><br>   --maxGaussians 6 \<br>   -resource:hapmap,VCF,known=false,training=true,truth=true,prior=15.0 hapmap_3.3.b37.sites.vcf \<br>   -resource:omni,VCF,known=false,training=true,truth=false,prior=12.0 1000G_omni2.5.b37.sites.vcf \<br>   -resource:dbsnp,VCF,known=true,training=false,truth=false,prior=6.0 dbsnp_135.b37.vcf \<br>   -an QD -an HaplotypeScore -an MQRankSum -an ReadPosRankSum -an FS -an MQ -an InbreedingCoeff \<br>   -mode SNP \<br></pre>
<p>Note that, for the above to work, the input vcf needs to be annotated with the corresponding values (QD, FS, MQ, etc.). If any of these values are somehow missing, then VariantAnnotator needs to be run first so that VariantRecalibration can run properly.<br></p><p>Also, note that some of these annotations might not be the best for your particular dataset. For example, InbreedingCoeff is a population level statistic that requires at least 10 samples in order to be calculated. Additionally, notice that DP was removed when working with hybrid capture datasets since there is extreme variation in the depth to which targets are captured. In whole genome experiments this variation is indicative of error but that is not the case in capture experiments.<br></p><p><b>Common ApplyRecalibration command</b><br></p><p>The <a rel="nofollow" href="/gsa/wiki/index.php/Variant_quality_score_recalibration" title="Variant quality score recalibration">Variant quality score recalibration</a> wiki page has an example command for the ApplyRecalibration step.<br></p>
<h5>Indel specific recommendations</h5>
<p>Arguments for <a rel="nofollow" href="/gsa/wiki/index.php/VariantFiltrationWalker" title="VariantFiltrationWalker">VariantFiltrationWalker</a>:<br></p>
<pre class="code codeBlock" spellcheck="false"><br>  --filterExpression "QD &lt; 2.0" \<br>  --filterExpression "ReadPosRankSum &lt; -20.0" \<br>  --filterExpression "InbreedingCoeff &lt; -0.8" \<br>  --filterExpression "FS &gt; 200.0" \<br>  --filterName QDFilter \<br>  --filterName ReadPosFilter \<br>  --filterName InbreedingFilter \<br>  --filterName FSFilter \<br></pre>
<p>Note the InbreedingCoeff statistic is a population-level calculation that is only available with 10 or more samples. If you have fewer samples you will need to omit that particular filter statement.<br></p>
<h3>Making analysis ready SNP and indel calls with hand filtering when VQSR is not possible</h3>
<p><a rel="nofollow" href="/gsa/wiki/index.php/Variant_quality_score_recalibration" title="Variant quality score recalibration">Variant quality score recalibration</a> requires a reasonable amount of data to work properly so with targeted resequencing of a small region (for example, a few hundred genes) VQSR may be inappropriate leaving hand filtering as the only option.  <br></p><p>Arguments for <a rel="nofollow" href="/gsa/wiki/index.php/VariantFiltrationWalker" title="VariantFiltrationWalker">VariantFiltrationWalker</a>:<br></p>
<pre class="code codeBlock" spellcheck="false"><br>  --filterExpression filter1<br>  --filterName filterName1<br>  --filterExpression filter2<br>  --filterName filterName2<br>  .<br>  .<br>  .<br></pre>
<p>where DATA_TYPE_SPECIFIC_FILTERS has project-specific filtering strings selected below:<br></p>
<ul><li> For SNPs
<ul><li> DATA_TYPE_SPECIFIC_FILTERS should be "QD &lt; 2.0", "MQ &lt; 40.0", "FS &gt; 60.0", "HaplotypeScore &gt; 13.0", "MQRankSum &lt; -12.5", "ReadPosRankSum &lt; -8.0".<br></li></ul></li></ul><ul><li> For Indels  
<ul><li> DATA_TYPE_SPECIFIC_FILTERS should be "QD &lt; 2.0", "ReadPosRankSum &lt; -20.0", "InbreedingCoeff &lt; -0.8", "FS &gt; 200.0".<br></li></ul></li></ul><p>Note that the InbreedingCoeff statistic is a population-level calculation that is only available with 10 or more samples. If you have fewer samples you will need to omit that particular filter statement.<br></p>
<ul><li> Shallow-coverage (&lt;10x) : you cannot use filtering to reliably separate TPs from FPs.  You must use the protocol involving <a rel="nofollow" href="/gsa/wiki/index.php/Variant_quality_score_recalibration" title="Variant quality score recalibration">Variant quality score recalibration</a><br></li></ul><p>The maximum DP (depth) filter only applies to whole genome data, where the probability of a site having exactly N reads given an average coverage of M is a well-behaved function.  First principles suggest this should be a binomial sampling but in practice it is more a Gaussian distribution.  Regardless, the DP threshold should be set a 5 or 6 sigma from the mean coverage across all samples, so that the DP &gt; X threshold eliminates sites with excessive coverage caused by alignment artifacts.  Note that <b>For exomes, a straight DP filter shouldn't be used</b> because the relationship between misalignments and depth isn't clear for capture data. <br></p><p>That said, all of the caveats about determining the right parameters, etc, are annoying and are largely eliminated by <a rel="nofollow" href="/gsa/wiki/index.php/Variant_quality_score_recalibration" title="Variant quality score recalibration">Variant quality score recalibration</a>.<br></p>
<h2>Expected SNP call quality</h2>
<p>All of the following tables were generated using <a rel="nofollow" href="http://gsa-stage:8080/gatkdocs/org_broadinstitute_sting_gatk_walkers_varianteval_VariantEvalWalker.html" title="VariantEval">VariantEval</a>.  You can generate your own data points for comparing with that tool and the correct input / comparison VCF files.<br></p>
<h3>Summary results for deep whole genome, multi-sample low-pass, and whole exome</h3>
<p>The associated table (see attachment) provides some expectations for running deep whole genomes, single whole exomes, and multi-sample low-pass (from the 1000 genomes) in terms of sensitivity, specificity, and Ti/Tv ratios for known and novel calls.  Obviously individual data sets will be different but this demonstrates that running the pipeline described here ( realigned -&gt; recalibration -&gt; calling -&gt; filtering -&gt; variant quality score recalibration) can produce excellent results for a variety of data types.  Note that the deep data sets are single sample; in our hands multi-sample deep data results in much better calls overall.<br></p>
<h3>Expected Ti/Tv ratios</h3>
<p>We provide two useful data points (see attachment) for evaluating the quality of SNP calls whole genome or in the targeted whole exome (Agilent).  You can follow a similar methodology to establish the Ti/Tv expectations for your region of interest, if captured separately, by running <a rel="nofollow" href="http://gsa-stage:8080/gatkdocs/org_broadinstitute_sting_gatk_walkers_varianteval_VariantEvalWalker.htm" title="VariantEval">VariantEval</a> on the 1000 Genomes Trios or Complete Genomes or other highly reliable data sets given the targeted interval.<br></p><p><br><br></p>