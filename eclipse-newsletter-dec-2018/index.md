# RasterFrames: Enabling DataFrame-Based Analysis of Big Spatiotemporal Raster Data

_Simeon H.K. Fitch_  
_VP of R&D_  
_Astraea, Inc._  

## Purpose

RasterFrames®, an incubating Eclipse Foundation LocationTech project, brings
together Earth-observing (EO) data analysis, big data computing,
and DataFrame-based data science. It was created to help data scientists and
analysts fully realize the rich potential of Earth-observing data. 

## Background

The recent explosion of EO data from public & private satellite operators
presents both a huge opportunity & challenge to the data analysis community. It
is Big Data in the truest sense. While EO & GIS specialists are
accustomed to working with this data, it is typically done so at a much less
expansive or global perspective. RasterFrames makes interactive analysis
possible on these large data sets without sacrificing accessibility.

## Why DataFames?

DataFrames are the _lingua franca_ of of data science. There's a long history
of organizing data in tabular form, with rows representing independent events or
observations, and columns representing measurements from that observation. From
agricultural records and transaction ledgers, to the advent of spreadsheets, and
on to the creation of [R Data Frames][R] and [Python Pandas][Pandas], this table-oriented data
structure remains a common and critical component of organizing data across
industries. This trend continues in the big data compute space with Apache
Spark SQL, which implements compute cluster distributed data frames.

## Architecture

RasterFrames extends the DataFrame abstraction for EO data on top of other
LocationTech projects: GeoTrellis, GeoMesa, JTS, & SFCurve. Through it the user
can perform raster analysis in SQL, Python, Java and Scala. 

![LocationTech Stack](rasterframes-locationtech-stack.png)

RasterFrames introduces a new native data type called `tile` to Spark SQL. A
"RasterFrame" is any DataFrame with one or more columns of type `tile`. A `tile`
column typically represents a single frequency band of sensor data, such as
"blue" or "near infrared", discretized into regular-sized chunks. Along with
`tile` column is typically a `geometry` column specifying the location of the
data, and a `timestamp` column representing the acquisition time.

![RasterFrame Anatomy](rasterframe-anatomy.png)

Raster data can be read from a number of sources. Through the flexible Spark SQL
DataSource API, RasterFrames can be constructed from collections of (preferably
Cloud Optimized) GeoTIFFs, GeoTrellis Layers, and from an experimental catalog
of Landsat 8 and MODIS data sets on AWS PDS. We are also experimenting with
support for the evolving [Spatiotemporal Asset Catalog (STAC)][STAC].

![RasterFrames data sources](rasterframes-data-sources.png)

## Example Use Case

In the following example we will attempt to provide a taste of the ...

In this example we will be using the [_MODIS Nadir BRDF-Adjusted Surface
Reflectance Data Product_][NBAR], which is directly available in [Amazon
Web Services (AWS) Public Data Set (PDS)][PDS]. We will be using the RasterFrames
MODIS catalog data source, and SQL as our language (as noted above, Python,
Java, and Scala are also options). 

The first step is to load our MODIS catalog data source into a table:

```sql
CREATE TEMPORARY VIEW modis USING `aws-pds-modis`;
DESCRIBE modis;
-- +----------------+------------------+
-- |col_name        |data_type         |
-- +----------------+------------------+
-- |product_id      |string            |
-- |acquisition_date|timestamp         |
-- |granule_id      |string            |
-- |gid             |string            |
-- |assets          |map<string,string>|
-- +----------------+------------------+
```

Next, we'll read in the red and NIR bands, globally, for a single day:

```sql
CREATE TEMPORARY VIEW red_nir_tiles AS
SELECT granule_id, rf_read_tiles(assets['B01'], assets['B02']) as (crs, extent, red, nir)
FROM modis
WHERE acquisition_date = to_timestamp('2013-01-04');
DESCRIBE red_nir_tiles;
-- +----------+-------------------------------------------------------+
-- |col_name  |data_type                                              |
-- +----------+-------------------------------------------------------+
-- |granule_id|string                                                 |
-- |crs       |struct<crsProj4:string>                                |
-- |extent    |struct<xmin:double,ymin:double,xmax:double,ymax:double>|
-- |red       |tile                                                   |
-- |nir       |tile                                                   |
-- +----------+-------------------------------------------------------+
```


Computing the [normalized difference vegetation index][NDVI] (NDVI) is a very
common operation in EO analysis, and is composed simply as the normalized
difference of the Red and NIR bands from a surface reflectance data product.

<!-- \text{NDVI} = \frac{\text{NIR} - \text{Red}}{\text{NIR} + \text{Red}} -->
![NDVI](ndvi.png)

Since a  normalized difference is a such common operation in EO analysis, RasterFrames
includes a single function for computing it:

```sql
CREATE TEMPORARY VIEW ndvi AS
SELECT rf_normalizedDifference(nir, red) as ndvi
FROM red_nir_tiles
```

However, suppose we want to compute something a bit more involved, such as the
EVI2 formula, as described in the [_MODIS VI Users Guide_][MODIS]:

<!-- \text{EVI2} = 2.5 \frac{\text{NIR} - \text{Red}}{\text{NIR} + 2.4 \, \text{Red} + 1} -->

![EVI2](evi2.png)

To do so we must make use of 

```sql
CREATE TEMPORARY VIEW evi2 AS
SELECT rf_localMultiplyScalar(
  rf_localDivide(
    rf_localSubtract(nir, red),
    rf_localAddScalar(
      rf_localAdd(nir, rf_localMultiplyScalar(red, 2.4)), 
      1.0
    )
  ), 2.5
) as EVI2
FROM red_nir_tiles
```




## Learning More

In-depth and more sophisticated examples, including clustering and
classification may be found on the RasterFrames website: rasterframes.io.

* rasterframes.io
* Jupyter Notebooks
* GitHub
* Gitter


[MODIS]:https://vip.arizona.edu/documents/MODIS/MODIS_VI_UsersGuide_June_2015_C6.pdf
[NBAR]:https://lpdaac.usgs.gov/dataset_discovery/modis/modis_products_table/mcd43a4_v006
[STAC]:https://github.com/radiantearth/stac-spec
[PDS]:https://registry.opendata.aws/modis/
[R]:https://www.rdocumentation.org/packages/base/versions/3.5.1/topics/data.frame
[Pandas]:https://pandas.pydata.org/
[NDVI]:https://en.wikipedia.org/wiki/Normalized_difference_vegetation_index