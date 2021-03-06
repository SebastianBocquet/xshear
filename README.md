xshear
======

Measure the tangential shear around a set of lenses.  This technique is also
known as cross-correlation shear, hence the name xshear

Runs of the code can easily be parallelized across sources as well as lenses.  

Parallelizing across sources makes sense for cross-correlation shear, because
the source catalog is generally much larger than the lens catalog. The source
catalog can be split into as many small chunks as needed, and each can be
processed on a different cpu across many machines.

Lenses can also be split up and the result simply concatenated.

Example Usage
-------------

```bash
# process the sources and lenses
cat source_file | xshear config_file lens_file > output_file

# you can parallelize by splitting up the sources.
cat sources1 | xshear config_file lens_file > output_file1
cat sources2 | xshear config_file lens_file > output_file2

# combine the lens sums from the different source catalogs using
# the redshear program (red for reduce: this is a map-reduce!).
cat output_file1 output_file2 | redshear config_file > output_file

# first apply a filter to a set of source files.
cat source_file | src_filter | xshear config_file lens_file > output_file

# The src_filter could be an awk command, etc.
cat source_file | awk '($6 > 0.2)' | xshear config_file lens_file > output_file
```

Example Config Files
---------------------

### Config File Using photoz as Truth
```python
# cosmology parameters
H0      = 100.0
omega_m = 0.25

# optional, nside for healpix, default 64
healpix_nside = 64

# masking style, for quadrant cuts. "none", "sdss", "equatorial"
mask_style = "equatorial"

# shear style
#  "reduced": normal reduced shear shapes, source catalog rows are like
#      ra dec g1 g2 weight ...
#  "lensfit": for lensfit with sensitivities
#      ra dec g1 g2 g1sens g2sens weight ...

shear_style = "reduced"

# sigma crit style
#  "point": point z for sources. Implies the last column in source cat is z
#  "interp": Interpolate 1/sigma_crit calculated from full P(z)).
#      Implies last N columns in source cat are 1/sigma_crit(zlens)_i

sigmacrit_style = "point"

# number of logarithmically spaced radial bins to use
nbin = 21

# min and max radius (units default to Mpc, see below)
rmin = 0.02
rmax = 35.15

# units of radius (Mpc or arcmin). If not set defaults to Mpc
r_units = "Mpc"

# units of shear, "shear" or "deltasig".  Defaults to deltasig.
shear_units = "deltasig"

# demand zs > zl + zdiff_min
# optional, only used for point z
zdiff_min       = 0.2
```

### Config File Using \Sigma_{crit}(zlens) Derived from Full P(zsource).   
```python
sigmacrit_style = "interp"

# zlens values for the 1/sigma_crit(zlens) values tabulated for each source
# note the elements of arrays can be separated by either spaces or commas
zlvals = [0.02 0.035 0.05 0.065 0.08 0.095 0.11 0.125 0.14 0.155 0.17 0.185 0.2 0.215 0.23 0.245 0.26 0.275 0.29 0.305 0.32 0.335 0.35 0.365 0.38 0.395 0.41]

```

### Alternative Units in Config Files

By default the code works in units of \Delta\Sigma (Msolar/pc^2) vs radius in
Mpc.  You can set the units of radius with "r_units" and the units for shear
with "shear_units"

To measure tangential shear vs radius in arcminutes
```
r_units     = "arcmin"
shear_units = "shear"
```
You can even measure \Delta\Sigma vs radius in arcminutes
```
r_units     = "arcmin"
shear_units = "deltasig"
```

Format of Lens Catalogs
-----------------------

The format is white-space delimited ascii.  The columns are

```
index ra dec z maskflags
```
For example:
```
0 239.5554049954774 27.24812220183897 0.09577668458223343 31
1 250.0985461117883 46.70283749181137 0.2325130850076675 31
2 197.8753117014942 -1.345204982754328 0.1821855753660202 7
...
```

The meaning of each column is
```
index:     a user-defined index
ra:        RA in degrees
dec:       DEC in degrees
z:         redshift
maskflags: flags for quadrant checking
```

The maskflags are only used if you set a mask style that is not "none" (no mask
flags)

Format of Source Catalogs
-----------------------
The format is white-space delimited ascii. The columns contained 
depend on the configuration.

When using point photozs (sigmacrit_style="point") the format is the following

```
For shear_style="reduced" (using simple reduced shear style)
        ra dec g1 g2 source_weight z

For shear_style="lensfit" (lensfit style)
        ra dec g1 g2 g1sens g2sens source_weight z
```

The format for sigmacrit_style="interp" includes the mean 1/sigma_crit in bins
of lens redshift.

```
For shear_style="reduced" (using simple reduced shear style)
        ra dec g1 g2 source_weight sc_1 sc_2 sc_3 sc_4 ...

For shear_style="lensfit" (lensfit style)
        ra dec g1 g2 g1sens g2sens source_weight sc_1 sc_2 sc_3 sc_4 ...
```

Meaning of columns:

```
ra:            RA in degrees
dec:           DEC in degrees
g1:            shape component 1
g2:            shape component 2
source_weight: a weight for the source
z:             a point estimator (when sigmacrit_style="point")
sc_i:          1/sigma_crit in bins of lens redshift.  The redshift bins
               are defined in "zlvals" config parameter
```

Format of Output Catalog
------------------------
The output catalog has a row for each lens.  For shear_style="reduced",
ordinary reduced shear style, each row looks like
```
    index weight_tot totpairs npair_i rsum_i wsum_i dsum_i osum_i
```

For shear style="lensfit", lensfit style
```
    index weight_tot totpairs npair_i rsum_i wsum_i dsum_i osum_i dsensum_i osensum_i
```

Where _i means radial bin i, so there will be Nbins columns for each.

Explanation for each of the output columns
```
index:      index from lens catalog
weight_tot: sum of all weights for all source pairs in all radial bins
totpairs:   total pairs used
npair_i:    number of pairs in radial bin i.  N columns.
rsum_i:     sum of radius in radial bin i
wsum_i:     sum of weights in radial bin i
dsum_i:     sum of \Delta\Sigma_+ * weights in radial bin i.
osum_i:     sum of \Delta\Sigma_x * weights in  radial bin i.
dsensum_i:  sum of weights times sensitivities
osensum_i:  sum of weights times sensitivities
```


In the above, the weights of each source are used as follows.  Let the weight
of a source be *weight_source* and the weight for a lens-source pair *w* be the
source weight times 1/\Sigma_{crit}^2 for that pair.  Then dsum_i is the sum
over sources j in radial bin i

```
w_j    = weight_source_j/\Sigma_{crit}^2
wsum_i = sum( w_j )
dsum_i = sum( w_j * \Delta\Sigma_+ )
\Delta\Sigma_+ = g+ * \Sigma_{crit}
```
For lensfit style, the sensitivities are also weighted
```
dsensum_i = sum( w_j * gsens )
```

### Averaging the \Delta\Sigma and Other Quantities

Just divide the columns.  For shear_style="reduced" (ordinary reduced shear)
```
    r = rsum_i/wsum_i
    \Delta\Sigma_+^i = dsum_i/wsum_i
    \Delta\Sigma_x^i = osum_i/wsum_i
```
For shear_style="lensfit", lensfit style
```
    \Delta\Sigma_+^i = dsum_i/dsensum_i
    \Delta\Sigma_x^i = osum_i/osensum_i
```

To average some other quantity associated with the lenses, use
the weights.  For example the mean redshift would be
```
sum( weight*z )/sum( weight )
```

compilation
-----------

```bash
# default build uses gcc
make

# use intel C compiler.
make CC=icc
```

install/uninstall
-----------------

```bash
make install prefix=/some/path
make uninstall prefix=/some/path
```

dependencies
------------

A C99 compliant compiler.
