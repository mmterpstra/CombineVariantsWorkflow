# CombineVariantsWorkflow
Small workflow for combining variants from multiple sources

# prerequirements
Install the https://github.com/mmterpstra/pipeline-util repo by installing the missing perl libraries (Vcftools perl library version 1.12b) and missing according to manual.

- run your favorite variant calling tools: freebayes, Gatk Haplotypecaller etc.
- Optionally run validateVariants to detect broken vcfs and fix / remove broken data 
1. run [CalleriseVcf.pl](https://github.com/mmterpstra/pipeline-util/blob/master/bin/CalleriseVcf.pl) to make everything traceable
2. merge the samples to a project level vcf for each caller using GenotypeGVCFs / Combinevariants
3. Run combinevariants on the project level VCF
4. This will remove AD/DP fields for complex snps and complex merges. Recover complex snp annotation from old data with [RecoverSampleAnnotationsAfterCombineVariants.pl](https://github.com/mmterpstra/pipeline-util/blob/master/bin/RecoverSampleAnnotationsAfterCombineVariants.pl). This will liftover the missing annotations from the old vcf as good as possible.
5. Regenotype the complex merges.
6. Recover the annotations from the regenotyping

# 1 Callerise

Flavoured annotatations eg sample.AD from haplotype caller also present as HC_sample.AD

```
CalleriseVcf.pl HC_ haplotypecaller.vcf > haplotypecaller.callerised.vcf
CalleriseVcf.pl FB_ freebayes.vcf > freebayes.callerised.vcf
CalleriseVcf.pl MuTect2_t_sampleX_n_sampleY MuTect2_t_sampleX_n_sampleY.vcf > MuTect2_t_sampleX_n_sampleY.callerised.vcf
CalleriseVcf.pl MuTect2_t_sampleZ_n_sampleY MuTect2_t_sampleZ_n_sampleY.vcf > MuTect2_t_sampleZ_n_sampleY.callerised.vcf

```
# 2 Optional merge by project

```
java -jar GenomeAnalysisToolkit.jar -T CombineVariants \
 --variant:SampleX MuTect2_t_sampleX_n_sampleY.callerised.vcf \
 --variant:SampleY MuTect2_t_sampleZ_n_sampleY.callerised.vcf \
 ...
 -o combine.MuTect2.vcf \
 -genotypeMergeOptions PRIORITIZE \
 -priority SampleX,SampleY \
 --filteredrecordsmergetype KEEP_UNCONDITIONAL
```

# 3 Merge different callers

This should lose some AD fields and the like for the complex variants (multiple alts). The complex merges also lose these.

```
java -Xmx4g -Djava.io.tmpdir=${variantCombineDir} \
  -XX:+UseConcMarkSweepGC  -XX:ParallelGCThreads=1 -jar $EBROOTGATK/GenomeAnalysisTK.jar \
 -T CombineVariants \
 -R ${onekgGenomeFasta} \
 --variant:GATK haplotyper.callerised.vcf \
 --variant:freebayes freebayes.callerised.vcf \
 --variant:MuTect2 mutect2.callerised.vcf \
 -o .tmp.combine.vcf \
 -genotypeMergeOptions PRIORITIZE \
 -priority GATK,freebayes,MuTect2 \
--filteredrecordsmergetype KEEP_UNCONDITIONAL
```

# 4  Recover annotations

Recover annotations from old vcf if chrom/pos/ref/alts are equal and write the complex merges [eg CHROM/POS/REF/ALT freebayes:1/100/AT/A + haplotypecaller 1/100/A/AG = 1/100/AT/A,AGT].

```
RecoverSampleAnnotationsAfterCombineVariants.pl \
complex.mergeout.vcf \
.tmp.combine.vcf \
haplotyper.callerised.vcf \
freebayes.callerised.vcf \
mutect2.callerised.vcf > .tmp.recover.vcf
```

# 5 Regenotype 
Regenotype the complex merges (in this case with HaplotypeCaller). Optional `--activeRegionMaxSize 200` for not missing variants and needed `--activeRegionExtension 200` for better assembly of larger variant calls than usually supported by GATK.

```
java -Xmx8g -Djava.io.tmpdir=${variantCombineDir}  -XX:+UseConcMarkSweepGC  -XX:ParallelGCThreads=1 -jar $EBROOTGATK/GenomeAnalysisTK.jar \
 -T HaplotypeCaller \
 -R ${onekgGenomeFasta} \
 --dbsnp ${dbsnpVcf}\
 $inputs \
 --activeRegionExtension 200 \
 --activeRegionMaxSize 200 \
 --genotyping_mode GENOTYPE_GIVEN_ALLELES \
 --alleles complex.mergeout.vcf \
 --output_mode EMIT_ALL_SITES \
 --forceActive \
 -stand_call_conf 0 \
 -L complex.mergeout.vcf \
 -o .tmp.complexHCregeno.vcf
 ```
 
# 6 Reannotate using regenotyped results
```
perl $EBROOTPIPELINEMINUTIL/bin/RecoverSampleAnnotationsAfterCombineVariants.pl \
 .tmp.ReallyComplex.vcf \
 .tmp.recover.vcf \
 .tmp.complexHCregeno.vcf \
> combine.vcf
```

# 7 interate / ValidateVCF

`.tmp.ReallyComplex.vcf` gives an open end and run variant validation tools to check your output.

# See also
 - Commit if you have improvements / alternate workflows
