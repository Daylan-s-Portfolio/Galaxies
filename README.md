JWST MIRI Composite Image Analysis
================
Daylan Salmeron
March 13th 2025

# Introduction

I have always been facinated with space. That is why I have chosen to
visualize **Galaxies IC 2163 and NGC 2207** located about 110 million
light years from Earth. This project processes three JWST MIRI FITS
images taken with filters **f770w**, **f1130w**, and **f1500w** . To
demonstrate my data science skills in R I have:

\- Visualized each filter individually.

\- Combine the 3 filters into a RGB Composite Plot.

\- Computed summary statistics.

\- Plot histograms of pixel intensities.

\- Evalated correlations between the filters.

\- Performed Principal Component Analysis (PCA) to explore dominant
variations. *PCA (Principal Component Analysis)* is a statistical
technique that transforms data into a set of orthogonal (uncorrelated)
components. The first principal component (PC1) captures the largest
amount of variance in the data, while subsequent components capture
decreasing amounts. This helps identify the underlying structure or
trends in complex datasets.

# 1. Load Libraries

``` r
knitr::opts_chunk$set(echo = TRUE, warning = FALSE, message = FALSE)
# These packages handle FITS file reading, data manipulation, plotting, interactive visualization, etc.}
library(FITSio)      # For reading FITS files
library(dplyr)       # For data manipulation
library(tidyr)       # For reshaping data
library(ggplot2)     # For static plotting
library(ggspatial)   # For adding scale bars and north arrows in plots
library(gridExtra)   # For arranging multiple ggplots together
library(corrplot)    # For plotting correlation matrices
```

# 2. Define Helper Functions

Below I defined several functions:

- **zscale_normalize**: Normalizes image data using the 5th and 95th
  percentiles.

- **extract_wcs**: Extracts basic WCS keywords (reference pixel, world
  coordinate, pixel scale) from a FITS header.

- **load_filter_data**: Loads a FITS image (and its WCS) for a given
  filter.

- **create_rgb_image**: Combines the three filter images into an RGB
  composite.

``` r
# Normalize image data: remove non-finite values, scale intensities between 0 and 1.}
zscale_normalize <- function(x, contrast = 0.25) {
  valid <- x[!is.na(x) & is.finite(x)]
  if (length(valid) < 2) return(matrix(0, nrow = nrow(x), ncol = ncol(x)))
  q <- quantile(valid, probs = c(0.05, 0.95))
  scaled <- (x - q[1]) / (q[2] - q[1]) * contrast
  pmax(pmin(scaled, 1), 0)
}

# Extract basic WCS info from a FITS header.
extract_wcs <- function(header) {
  required <- c("CRPIX1", "CRPIX2", "CDELT1", "CDELT2", "CRVAL1", "CRVAL2")
  values <- sapply(required, function(key) {
    idx <- which(header == key)
    if (length(idx) == 0) stop("Missing WCS header: ", key)
    as.numeric(header[idx + 1])
  })
  names(values) <- required
  # Convert CDELT from degrees per pixel to arcsec per pixel (1° = 3600 arcsec)
  values["CDELT1"] <- values["CDELT1"] * 3600
  values["CDELT2"] <- values["CDELT2"] * 3600
  as.list(values)
}

# Load a filter's FITS image and extract its WCS information.
load_filter_data <- function(filter_config) {
  fits_file <- file.path(filter_config$path,
                         paste0("jw06553-o001_t001_miri_", filter_config$name, "_i2d.fits"))
  cat_file <- file.path(filter_config$path,
                        paste0("jw06553-o001_t001_miri_", filter_config$name, "_cat.ecsv"))

  
  fits <- readFITS(fits_file)
  wcs <- extract_wcs(fits$hdr)
  
  list(
    name = filter_config$name,
    image = fits$imDat,
    wcs = wcs,
    color = filter_config$color
  )
}

# Create an RGB composite image from three filters.
create_rgb_image <- function(data_list, gamma = 0.4, contrast = 0.3) {
  process_channel <- function(img) {
    # Replace NA values with the median, then normalize and apply gamma correction.
    img[is.na(img)] <- median(img, na.rm = TRUE)
    zscale_normalize(img, contrast) ^ gamma
  }
  
  # Create a 3D array to hold the red, green, and blue channels.
  rgb_stack <- array(dim = c(dim(data_list[[1]]$image), 3))
  rgb_stack[,,1] <- process_channel(data_list$f1500w$image)  # Red channel
  rgb_stack[,,2] <- process_channel(data_list$f1130w$image)  # Green channel
  rgb_stack[,,3] <- process_channel(data_list$f770w$image)   # Blue channel
  
  # Convert the 3D image array to a data frame for plotting.
  df <- expand.grid(x = 1:dim(rgb_stack)[1], y = 1:dim(rgb_stack)[2]) %>% 
    mutate(
      R = as.vector(rgb_stack[,,1]),
      G = as.vector(rgb_stack[,,2]),
      B = as.vector(rgb_stack[,,3]),
      hex = rgb(R, G, B)
    )
  
  # Build the ggplot with a reversed y-axis, scale bar, and north arrow.
  plt <- ggplot(df, aes(x = x, y = y)) +
    geom_raster(fill = df$hex) +
    scale_y_reverse() +
    annotation_scale(location = "br") +
    annotation_north_arrow(location = "tr") +
    theme_void() +
    labs(title = "JWST MIRI Enhanced Composite",
         subtitle = paste("Gamma:", gamma, "Contrast:", contrast))
  
  return(plt)
}
```

# 3. Define Filter Configurations

We specify the file paths and color codes for each filter image.

``` r
filters <- list(
  list(name = "f770w", path = "Data/MAST_2025-03-09T1739 2/JWST/jw06553-o001_t001_miri_f770w", color = "#377EB8"),
  list(name = "f1130w", path = "Data/MAST_2025-03-09T1739 3/JWST/jw06553-o001_t001_miri_f1130w", color = "#4DAF4A"),
  list(name = "f1500w", path = "Data/MAST_2025-03-09T1739/JWST/jw06553-o001_t001_miri_f1500w", color = "#E41A1C")
)
```

# 4. Load Data from All Filters

I looped through the filters and load the FITS data along with WCS
information.

``` r
filter_data <- lapply(filters, load_filter_data)
names(filter_data) <- sapply(filter_data, function(x) x$name)
```

# 5. Visualize Each Filter Individually

Before creating the composite image, I displayed each filter’s image
individually in a row.

``` r
# Define color scales using correct JWST filter colors
color_f770w <- c("black", "#377EB8", "lightblue")  # Blue for F770W
color_f1130w <- c("black", "#4DAF4A", "lightgreen")  # Green for F1130W
color_f1500w <- c("black", "#E41A1C", "pink")  # Red for F1500W

# Function to plot filter images with assigned colors
plot_filter_image <- function(filt, gamma = 0.8, contrast = 0.1, color_scale) {
  process_channel <- function(img) {
    img[is.na(img)] <- median(img, na.rm = TRUE)
    zscale_normalize(img, contrast) ^ gamma
  }
  
  img_norm <- process_channel(filt$image)
  
  df <- expand.grid(x = 1:dim(img_norm)[1], y = 1:dim(img_norm)[2]) %>%
    mutate(intensity = as.vector(img_norm))
  
  ggplot(df, aes(x = x, y = y, fill = intensity)) +
    geom_raster() +
    scale_fill_gradientn(colors = color_scale) +  # Apply correct filter color
    scale_y_reverse() +
    theme_void() +
    labs(title = paste("JWST MIRI -", filt$name)) +
    theme(legend.position = "bottom")  # Move legend to the bottom
}

# Generate individual plots with correct colors
p_f770w <- plot_filter_image(filter_data$f770w, gamma = 0.8, contrast = 0.1, color_f770w)
p_f1130w <- plot_filter_image(filter_data$f1130w, gamma = 0.8, contrast = 0.1, color_f1130w)
p_f1500w <- plot_filter_image(filter_data$f1500w, gamma = 0.8, contrast = 0.1, color_f1500w)


print("Pixel intensity by Filter")
```

    ## [1] "Pixel intensity by Filter"

``` r
# Arrange them side-by-side
grid.arrange(p_f770w, p_f1130w, p_f1500w, ncol = 3)
```

![](galaxies_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

# 6. Create the Final RGB Composite Plot

Now I combined the three filters into a single RGB composite image and
save the result.

``` r
  final_plot <- create_rgb_image(filter_data, gamma = 0.8, contrast = 0.1)
  
  print(final_plot)
```

![](galaxies_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

# 7. Statistical Analysis

## 7.1 Summary Statistics

I computed basic statistics (mean, median, standard deviation) for pixel
intensities for each filter.

``` r
compute_stats <- function(img) {
  list(mean = mean(img, na.rm = TRUE),
       median = median(img, na.rm = TRUE),
       sd = sd(img, na.rm = TRUE))
}

stats_list <- lapply(filter_data, function(filt) compute_stats(filt$image))
stats_df <- do.call(rbind, lapply(names(stats_list), function(n) {
  data.frame(filter = n, t(as.data.frame(stats_list[[n]])))
}))
print("Summary Statistics for each filter:")
```

    ## [1] "Summary Statistics for each filter:"

``` r
print(stats_df)
```

    ##         filter t.as.data.frame.stats_list..n....
    ## mean     f770w                          4.922735
    ## median   f770w                          4.195947
    ## sd       f770w                          4.329506
    ## mean1   f1130w                         18.150068
    ## median1 f1130w                         17.239799
    ## sd1     f1130w                          7.027549
    ## mean2   f1500w                         42.560256
    ## median2 f1500w                         42.077408
    ## sd2     f1500w                          6.898058

**Summary Statistics:**

- **f770w:** Mean ≈ 4.92, Median ≈ 4.20, SD ≈ 4.33  

  *Indicates that this filter generally has lower intensity values with
  moderate variability.*

- **f1130w:** Mean ≈ 18.15, Median ≈ 17.24, SD ≈ 7.03  

  *Shows intermediate brightness with a larger spread compared to
  f770w.*

- **f1500w:** Mean ≈ 42.56, Median ≈ 42.08, SD ≈ 6.90  

  *This filter is significantly brighter on average with a similar level
  of spread as f1130w.*

The differences in mean and median values reflect that the f1500w filter
captures stronger signals, while f770w registers much fainter emission.
The variations (standard deviations) show how spread out the pixel
intensities are in each filter.

## 7.2 Histograms of Pixel Intensities

Histograms help us understand the distribution of pixel values for each
filter.When pixel intensities span several orders of magnitude and are
mostly near zero, applying a log transform makes the distribution more
interpretable:

``` r
plot_log_histogram <- function(img, filter_name) {
  df <- data.frame(intensity = as.vector(img))
  
  ggplot(df, aes(x = intensity)) +
    # Transform the x-axis to log scale
    geom_histogram(bins = 50, fill = "steelblue", color = "black") +
    scale_x_log10() +            # <-- log scale on the x-axis
    theme_minimal() +
    labs(
      title = paste("Log-Scale Histogram -", filter_name),
      x = "Pixel Intensity (log scale)",
      y = "Count"
    )
}

p_log_f770w <- plot_log_histogram(filter_data$f770w$image, "f770w")
p_log_f1130w <- plot_log_histogram(filter_data$f1130w$image, "f1130w")
p_log_f1500w <- plot_log_histogram(filter_data$f1500w$image, "f1500w")

grid.arrange(p_log_f770w, p_log_f1130w, p_log_f1500w, ncol = 1)
```

![](galaxies_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

1.  **Highly Skewed Distributions**

    - Most pixels still have intensities near or slightly above zero,
      but now we can see more structure in the low‐value region because
      the log scale stretches that region.

    - The bars around ~1–10 on the log scale are tall, indicating that
      the majority of pixels have intensities in that range.

2.  **Differences Between Filters**

    - **f770w**: The histogram peak is around lower intensities (log
      scale 1–5).

    - **f1130w**: Shifts a bit higher (peak around log scale 10–20).

    - **f1500w**: Even further to the right, with intensities often \>30
      on the log scale.

3.  **Tiny Tail at High Values**

    - The extreme right side of each histogram (e.g., intensities of
      1000+ on the log scale) is nearly zero in height, meaning only a
      very small fraction of pixels are that bright.

4.  **Better Visibility of Bulk**

    - Because I used a log transform, I avoid having one huge bar at
      zero and a nearly empty plot for the rest. Instead, we can see how
      intensities spread over a few orders of magnitude, revealing that
      the bulk is within 1–100 on the log scale, and only rare pixels
      exceed 100 or 1000.

In short, **the log‐scale histograms** give us a more informative view
of the data’s distribution. Instead of having a giant spike near zero,
we see that there’s a range of intensities that’s more distinguishable
once we compress the higher values and expand the lower ones via a
logarithmic transformation.

# 8. Correlation Analysis

I combined the pixel data from all filters into a data frame and compute
a correlation matrix to see how similar the images are across filters.

``` r
df_corr <- data.frame(
  f770w = as.vector(filter_data$f770w$image),
  f1130w = as.vector(filter_data$f1130w$image),
  f1500w = as.vector(filter_data$f1500w$image)
)
corr_matrix <- cor(df_corr, use = "complete.obs")
print("Correlation matrix between filters:")
```

    ## [1] "Correlation matrix between filters:"

``` r
print(corr_matrix)
```

    ##            f770w    f1130w    f1500w
    ## f770w  1.0000000 0.9533836 0.7177089
    ## f1130w 0.9533836 1.0000000 0.8442387
    ## f1500w 0.7177089 0.8442387 1.0000000

``` r
corrplot(corr_matrix, method = "circle")
```

![](galaxies_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

**Correlation Matrix:**

- **f770w and f1130w** are very strongly correlated (r ≈ 0.95),
  indicating that these filters record very similar intensity patterns.

- **f1130w and f1500w** are also strongly correlated (r ≈ 0.84), though
  slightly less so.

- **f770w and f1500w** show a moderately strong correlation (r ≈ 0.72),
  suggesting some differences between the shortest and longest
  wavelength channels.

Overall, the three filters measure similar structures, but differences
(likely due to wavelength-specific features) become apparent in the
slightly lower correlations between f770w and f1500w.

# 9. Principal Component Analysis (PCA)

PCA is used to reduce the complexity of the data and reveal the main
trends. Here, it tells us if most of the variance is shared across
filters (e.g., overall brightness) or if there are secondary
differences.

``` r
# Remove rows with missing or infinite values to avoid PCA errors.
df_corr_clean <- df_corr[complete.cases(df_corr) & apply(df_corr, 1, function(row) all(is.finite(row))), ]
pca_res <- prcomp(df_corr_clean, center = TRUE, scale. = TRUE)
print("PCA Summary:")
```

    ## [1] "PCA Summary:"

``` r
print(summary(pca_res))
```

    ## Importance of components:
    ##                           PC1     PC2    PC3
    ## Standard deviation     1.6372 0.54411 0.1529
    ## Proportion of Variance 0.8935 0.09869 0.0078
    ## Cumulative Proportion  0.8935 0.99220 1.0000

``` r
pca_df <- as.data.frame(pca_res$x)
pca_plot <- ggplot(pca_df, aes(x = PC1, y = PC2)) +
  geom_point(alpha = 0.2, color = "darkblue") +
  theme_minimal() +
  labs(title = "PCA of Pixel Intensities",
       x = "PC1", y = "PC2")
print(pca_plot)
```

![](galaxies_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

**Principal Component Analysis (PCA):**

- **PC1** has a standard deviation of 1.6372 and explains about
  **89.35%** of the variance.

- **PC2** explains roughly **9.87%** of the variance, while **PC3**
  contributes only about **0.78%**.

- The cumulative variance of PC1 and PC2 is approximately **99.22%**.

Most of the variation across pixels is captured by one dominant
component (likely overall brightness). The second component captures
minor, secondary variations between the filters, while the third
component is negligible.

# Conclusion

In this project, I:

Loaded JWST MIRI FITS images from three filters.

Visualized individual filter images and combined them into an RGB
composite.

Computed summary statistics and generated histograms to understand pixel
intensity distributions.

Analyzed the correlation between filters. Performed PCA to reveal
dominant trends in the data.

This analysis demonstrates proficiency in R programming, image
processing, and data science techniques applied to astrophysical data.
