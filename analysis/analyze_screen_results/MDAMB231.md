Screen Results - MDAMB231
================
Matthew Berginski

Results of the PTEN-KO vs WT siRNA screen. Each plate was normalized to
the median of the three replicates. Then the cell titer glo values were
scaled to the range between mean of the two siRNA controls on a
per-plate basis.

# Variation in Per Plate siRNA Controls

![](MDAMB231_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

# Per siRNA Target WT vs PTEN-KO Cell Growth/Viability

![](MDAMB231_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

# Per siRNA Target WT vs PTEN-KO Cell Growth/Viability with 95% Conf Interval

I’ve added 95% confidence intervals (represented by lines in the x/y
directions) for each siRNA treatment.

This figure is way to noisy to make out anything about the individual
siRNA treatments, but I think it’s helpful to get a grasp on how much
variability there is in the plate replicates.

![](MDAMB231_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

# Histogram of the Difference in Viability/Growth for each siRNA treatment

Same data as above except represented as the difference between KO/WT.
The average difference is 0.0117.

![](MDAMB231_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

# Searching for Significant Differences in Cell Viability

I took the cell viability results and ran a t-test on all the WT vs KO
values. I adjusted the resulting p-values for multiple testing and
filtered to the 0.05 level. There were 44 total hits. The following
table lists the hits, ordered by the genes with the strongest
differential affect on the KO cells.

<table>

<thead>

<tr>

<th style="text-align:left;">

hgnc\_symbol

</th>

<th style="text-align:right;">

WT\_mean\_frac

</th>

<th style="text-align:right;">

KO\_mean\_frac

</th>

<th style="text-align:right;">

KO\_change\_diff

</th>

<th style="text-align:right;">

p.value\_adj

</th>

</tr>

</thead>

<tbody>

<tr>

<td style="text-align:left;">

VPS37A

</td>

<td style="text-align:right;">

2.430

</td>

<td style="text-align:right;">

1.300

</td>

<td style="text-align:right;">

\-1.1330

</td>

<td style="text-align:right;">

0.01457

</td>

</tr>

<tr>

<td style="text-align:left;">

FAM83H

</td>

<td style="text-align:right;">

1.490

</td>

<td style="text-align:right;">

1.000

</td>

<td style="text-align:right;">

\-0.4894

</td>

<td style="text-align:right;">

0.04518

</td>

</tr>

<tr>

<td style="text-align:left;">

SGIP1

</td>

<td style="text-align:right;">

1.590

</td>

<td style="text-align:right;">

1.170

</td>

<td style="text-align:right;">

\-0.4193

</td>

<td style="text-align:right;">

0.03289

</td>

</tr>

<tr>

<td style="text-align:left;">

PRH1

</td>

<td style="text-align:right;">

1.270

</td>

<td style="text-align:right;">

0.903

</td>

<td style="text-align:right;">

\-0.3625

</td>

<td style="text-align:right;">

0.04596

</td>

</tr>

<tr>

<td style="text-align:left;">

MITD1

</td>

<td style="text-align:right;">

1.600

</td>

<td style="text-align:right;">

1.270

</td>

<td style="text-align:right;">

\-0.3295

</td>

<td style="text-align:right;">

0.02274

</td>

</tr>

<tr>

<td style="text-align:left;">

NUMBL

</td>

<td style="text-align:right;">

1.260

</td>

<td style="text-align:right;">

0.966

</td>

<td style="text-align:right;">

\-0.2988

</td>

<td style="text-align:right;">

0.04468

</td>

</tr>

<tr>

<td style="text-align:left;">

CDC42

</td>

<td style="text-align:right;">

1.390

</td>

<td style="text-align:right;">

1.100

</td>

<td style="text-align:right;">

\-0.2908

</td>

<td style="text-align:right;">

0.01457

</td>

</tr>

<tr>

<td style="text-align:left;">

EFNA1

</td>

<td style="text-align:right;">

1.010

</td>

<td style="text-align:right;">

0.776

</td>

<td style="text-align:right;">

\-0.2298

</td>

<td style="text-align:right;">

0.04929

</td>

</tr>

<tr>

<td style="text-align:left;">

IER2

</td>

<td style="text-align:right;">

0.750

</td>

<td style="text-align:right;">

0.948

</td>

<td style="text-align:right;">

0.1977

</td>

<td style="text-align:right;">

0.02274

</td>

</tr>

<tr>

<td style="text-align:left;">

U2AF2

</td>

<td style="text-align:right;">

0.479

</td>

<td style="text-align:right;">

0.684

</td>

<td style="text-align:right;">

0.2057

</td>

<td style="text-align:right;">

0.03289

</td>

</tr>

<tr>

<td style="text-align:left;">

MAP3K6

</td>

<td style="text-align:right;">

0.767

</td>

<td style="text-align:right;">

0.984

</td>

<td style="text-align:right;">

0.2171

</td>

<td style="text-align:right;">

0.03289

</td>

</tr>

<tr>

<td style="text-align:left;">

BMPER

</td>

<td style="text-align:right;">

0.461

</td>

<td style="text-align:right;">

0.690

</td>

<td style="text-align:right;">

0.2294

</td>

<td style="text-align:right;">

0.03731

</td>

</tr>

<tr>

<td style="text-align:left;">

BCAR1

</td>

<td style="text-align:right;">

0.348

</td>

<td style="text-align:right;">

0.579

</td>

<td style="text-align:right;">

0.2313

</td>

<td style="text-align:right;">

0.01457

</td>

</tr>

<tr>

<td style="text-align:left;">

OR4Q3

</td>

<td style="text-align:right;">

0.736

</td>

<td style="text-align:right;">

0.979

</td>

<td style="text-align:right;">

0.2429

</td>

<td style="text-align:right;">

0.04969

</td>

</tr>

<tr>

<td style="text-align:left;">

CLSPN

</td>

<td style="text-align:right;">

0.604

</td>

<td style="text-align:right;">

0.894

</td>

<td style="text-align:right;">

0.2898

</td>

<td style="text-align:right;">

0.04929

</td>

</tr>

<tr>

<td style="text-align:left;">

RNF133

</td>

<td style="text-align:right;">

1.150

</td>

<td style="text-align:right;">

1.440

</td>

<td style="text-align:right;">

0.2909

</td>

<td style="text-align:right;">

0.04969

</td>

</tr>

<tr>

<td style="text-align:left;">

MRPL42

</td>

<td style="text-align:right;">

0.315

</td>

<td style="text-align:right;">

0.609

</td>

<td style="text-align:right;">

0.2940

</td>

<td style="text-align:right;">

0.04100

</td>

</tr>

<tr>

<td style="text-align:left;">

SNRPC

</td>

<td style="text-align:right;">

0.495

</td>

<td style="text-align:right;">

0.810

</td>

<td style="text-align:right;">

0.3152

</td>

<td style="text-align:right;">

0.03731

</td>

</tr>

<tr>

<td style="text-align:left;">

PTCH2

</td>

<td style="text-align:right;">

0.362

</td>

<td style="text-align:right;">

0.680

</td>

<td style="text-align:right;">

0.3181

</td>

<td style="text-align:right;">

0.04753

</td>

</tr>

<tr>

<td style="text-align:left;">

S1PR2

</td>

<td style="text-align:right;">

0.492

</td>

<td style="text-align:right;">

0.814

</td>

<td style="text-align:right;">

0.3217

</td>

<td style="text-align:right;">

0.04834

</td>

</tr>

<tr>

<td style="text-align:left;">

NANS

</td>

<td style="text-align:right;">

0.300

</td>

<td style="text-align:right;">

0.637

</td>

<td style="text-align:right;">

0.3371

</td>

<td style="text-align:right;">

0.03321

</td>

</tr>

<tr>

<td style="text-align:left;">

SLC25A2

</td>

<td style="text-align:right;">

0.472

</td>

<td style="text-align:right;">

0.813

</td>

<td style="text-align:right;">

0.3402

</td>

<td style="text-align:right;">

0.03321

</td>

</tr>

<tr>

<td style="text-align:left;">

FKBP4

</td>

<td style="text-align:right;">

0.776

</td>

<td style="text-align:right;">

1.120

</td>

<td style="text-align:right;">

0.3448

</td>

<td style="text-align:right;">

0.03397

</td>

</tr>

<tr>

<td style="text-align:left;">

ATM

</td>

<td style="text-align:right;">

0.691

</td>

<td style="text-align:right;">

1.040

</td>

<td style="text-align:right;">

0.3480

</td>

<td style="text-align:right;">

0.02401

</td>

</tr>

<tr>

<td style="text-align:left;">

COPA

</td>

<td style="text-align:right;">

0.207

</td>

<td style="text-align:right;">

0.563

</td>

<td style="text-align:right;">

0.3562

</td>

<td style="text-align:right;">

0.03289

</td>

</tr>

<tr>

<td style="text-align:left;">

CCDC158

</td>

<td style="text-align:right;">

0.206

</td>

<td style="text-align:right;">

0.565

</td>

<td style="text-align:right;">

0.3589

</td>

<td style="text-align:right;">

0.01457

</td>

</tr>

<tr>

<td style="text-align:left;">

SRSF6

</td>

<td style="text-align:right;">

0.621

</td>

<td style="text-align:right;">

0.986

</td>

<td style="text-align:right;">

0.3646

</td>

<td style="text-align:right;">

0.02274

</td>

</tr>

<tr>

<td style="text-align:left;">

VPS36

</td>

<td style="text-align:right;">

0.141

</td>

<td style="text-align:right;">

0.507

</td>

<td style="text-align:right;">

0.3659

</td>

<td style="text-align:right;">

0.03262

</td>

</tr>

<tr>

<td style="text-align:left;">

PIGK

</td>

<td style="text-align:right;">

0.653

</td>

<td style="text-align:right;">

1.030

</td>

<td style="text-align:right;">

0.3736

</td>

<td style="text-align:right;">

0.03731

</td>

</tr>

<tr>

<td style="text-align:left;">

MANF

</td>

<td style="text-align:right;">

0.488

</td>

<td style="text-align:right;">

0.863

</td>

<td style="text-align:right;">

0.3753

</td>

<td style="text-align:right;">

0.03038

</td>

</tr>

<tr>

<td style="text-align:left;">

PRC1

</td>

<td style="text-align:right;">

0.279

</td>

<td style="text-align:right;">

0.655

</td>

<td style="text-align:right;">

0.3761

</td>

<td style="text-align:right;">

0.02274

</td>

</tr>

<tr>

<td style="text-align:left;">

DDX41

</td>

<td style="text-align:right;">

0.434

</td>

<td style="text-align:right;">

0.810

</td>

<td style="text-align:right;">

0.3761

</td>

<td style="text-align:right;">

0.04207

</td>

</tr>

<tr>

<td style="text-align:left;">

STK35

</td>

<td style="text-align:right;">

0.577

</td>

<td style="text-align:right;">

0.964

</td>

<td style="text-align:right;">

0.3868

</td>

<td style="text-align:right;">

0.04834

</td>

</tr>

<tr>

<td style="text-align:left;">

MYBL2

</td>

<td style="text-align:right;">

0.196

</td>

<td style="text-align:right;">

0.590

</td>

<td style="text-align:right;">

0.3936

</td>

<td style="text-align:right;">

0.02274

</td>

</tr>

<tr>

<td style="text-align:left;">

CCAR2

</td>

<td style="text-align:right;">

0.531

</td>

<td style="text-align:right;">

0.938

</td>

<td style="text-align:right;">

0.4075

</td>

<td style="text-align:right;">

0.02932

</td>

</tr>

<tr>

<td style="text-align:left;">

IMPACT

</td>

<td style="text-align:right;">

0.337

</td>

<td style="text-align:right;">

0.759

</td>

<td style="text-align:right;">

0.4219

</td>

<td style="text-align:right;">

0.02274

</td>

</tr>

<tr>

<td style="text-align:left;">

HK2

</td>

<td style="text-align:right;">

0.379

</td>

<td style="text-align:right;">

0.825

</td>

<td style="text-align:right;">

0.4464

</td>

<td style="text-align:right;">

0.03289

</td>

</tr>

<tr>

<td style="text-align:left;">

MOCS3

</td>

<td style="text-align:right;">

0.437

</td>

<td style="text-align:right;">

0.887

</td>

<td style="text-align:right;">

0.4504

</td>

<td style="text-align:right;">

0.03289

</td>

</tr>

<tr>

<td style="text-align:left;">

LRRC39

</td>

<td style="text-align:right;">

0.242

</td>

<td style="text-align:right;">

0.700

</td>

<td style="text-align:right;">

0.4584

</td>

<td style="text-align:right;">

0.04518

</td>

</tr>

<tr>

<td style="text-align:left;">

SRP54

</td>

<td style="text-align:right;">

0.442

</td>

<td style="text-align:right;">

0.914

</td>

<td style="text-align:right;">

0.4715

</td>

<td style="text-align:right;">

0.04834

</td>

</tr>

<tr>

<td style="text-align:left;">

OR4K17

</td>

<td style="text-align:right;">

0.477

</td>

<td style="text-align:right;">

0.954

</td>

<td style="text-align:right;">

0.4772

</td>

<td style="text-align:right;">

0.04100

</td>

</tr>

<tr>

<td style="text-align:left;">

EIF3K

</td>

<td style="text-align:right;">

0.586

</td>

<td style="text-align:right;">

1.060

</td>

<td style="text-align:right;">

0.4792

</td>

<td style="text-align:right;">

0.01457

</td>

</tr>

<tr>

<td style="text-align:left;">

QDPR

</td>

<td style="text-align:right;">

0.513

</td>

<td style="text-align:right;">

0.999

</td>

<td style="text-align:right;">

0.4867

</td>

<td style="text-align:right;">

0.03038

</td>

</tr>

<tr>

<td style="text-align:left;">

TXN2

</td>

<td style="text-align:right;">

0.470

</td>

<td style="text-align:right;">

1.040

</td>

<td style="text-align:right;">

0.5705

</td>

<td style="text-align:right;">

0.04969

</td>

</tr>

</tbody>

</table>

# A Few Visualizations of the Hit Table

Same style figure as before, but filtered to only the hits in the above
table.

![](MDAMB231_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

Same figure as above, but with labels for each gene and colored the
points/labels by the strength of the KO vs WT effect.

![](MDAMB231_files/figure-gfm/unnamed-chunk-12-1.png)<!-- -->

A small multiples visualization of each gene hit, in order by the
differential effect on the KO cells.

![](MDAMB231_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->
