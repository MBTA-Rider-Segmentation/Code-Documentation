---
nav_include: 1
title: Module Class Documentation
---

## Contents
{:.no_toc}
*  
{: toc}

## Feature Extraction

The ```feature.py``` module cleans MBTA transaction-level data and extracts rider-level pattern-of-use features.

### Class ```DataLoader```

A ```DataLoader``` object first merges the joint AFC_ODX table, the stops table and fare product tables to form transaction records. The preprocessed transaction records are then passed to a ```FeatureExtractor``` object to extract the rider-level pattern-of-use features.

Note: A ```DataLoader``` object is initialized by a ```FeatureExtractor``` object and is not explicitly used elsewhere in our project.

- **Attributes**:
  - ```start_month```: a string representing the start month in the format of YYMM, e.g. '1710'
  - ```duration```: an integer representing the length of duration (in months)
  - ```afc_odx_fields```: a list of fields used to read the joint AFC_ODX table, ['deviceclassid', 'trxtime', 'tickettypeid', 'card', 'origin', 'movementtype']
  - ```fp_field```: a list of fields used to read the fare product table, ['tariff', 'servicebrand', 'usertype', 'tickettypeid', 'zonecr']
  - ```stops_fields```: a list of fields used to read the stops table, ['stop_id', 'zipcode']
  - ```fareprod```: a DataFrame of fare product records
  - ```stops```: a DataFrame of stop records
  - ```station_deviceclassid```: a list of device class IDs of interest, [411, 412, 441, 442, 443, 501, 503]
  - ```validation_movementtype```: a list of validation movement types of interest, [7, 20]
  - ```df```: a DataFrame of preprocessed transaction records

- **Methods**:
  - ```__init__(self, start_month, duration)```:
    - initialize the attributes
    - read in the stops and the fare product table as DataFrames

  - ```load(self)```:
    - read in the joint AFC_ODX table corresponding to the specified ```start_month``` and ```duration``` as a DataFrame
    - merge joint AFC_ODX, stops and fare product table and select rows with ```station_deviceclassid``` and ```validation_movementtype``` of interest
    - save the preprocessed transaction records as ```self.df```

### Class ```FeatureExtractor```

A ```FeatureExtractor``` object extracts the rider-level temporal, geographical, and ticket-purchasing features based on the preprocessed transaction records returned by the ```DataLoader```
  Label riders by their total number of trips, and whether they use commuter rail expect for zone 1a
  The second step is for further filtering in segmentation model.

- **Attributes**:
  - ```start_month```: a string representing the start month in the format of YYMM, e.g. '1710'
  - ```duration```: an integer representing the length of duration
  - ```df_transaction```: a DataFrame of preprocessed transaction records returned by a ```DataLoader``` object
  - ```purchase_features```: a list of ticket-purchasing types, ['tariff', 'usertype', 'servicebrand', 'zonecr']

- **Methods**:
  - ```__init__(self, start_month='1701', duration=1)```:
    - initialize the attributes
    - get the preprocessed transaction DataFrame by initializing a ```DataLoader``` object

  - ```_extract_temporal_patterns(self)```: For each ```riderID```,
    - extract the [168 hourly temporal patterns](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#feature-set-1a-168-hourly-version), each with a column name prefix 'hr_'
    - extract the [weekend-vs-weekday trip counts](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#feature-set-2-weekday-vs-weekend-total-counts), with column name 'weekday' and 'weekend'
    - extract the [48 weekday hourly vs. weekend hourly temporal patterns](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#feature-set-1b-48-weekday-hourly-vs-weekend-hourly-version), each with a column name prefix 'wkday_24_'
    - extract the [time flexibility score](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#feature-set-3-time-flexibility-score), with column name 'flex_wkday_24' and 'flex_wkend_24'
    - extract the [most frequent trip hours](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#feature-set-4-most-frequent-trip-hours), with column name 'max_wkday_24_1', 'max_wkday_24_2', 'max_wkend_24_1'
    - return all sets of temporal features as a DataFrame

  - ```_extract_geographical_patterns(self)```:
    - extract the [# trips originated in each zip code](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#geographical-patterns), each with a column name prefix 'zipcode_'
    - return the geographical features as a DataFrame

  - ```_get_one_purchase_feature(self, feature)```:
    - argument ```feature```: one item from the ```purchase_features``` list
    - extract the [# trips associated with the ticket purchasing dimension specified in ```feature```](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#ticket-purchasing-pattern), each with a column name prefix '{feature}_'
    - return one type of ticket purchasing features as a DataFrame

  - ```_extract_ticket_purchasing_patterns(self)```:
    - call ```_get_one_purchase_feature(self, feature)``` to extract the [number of trips associated with all ticket purchasing dimensions specified in ```purchase_features```](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#ticket-purchasing-pattern)
    - return the ticket-purchasing features as a DataFrame

  - ```_label_rider_by_trip_frequency(self, rider)```:
    - argument ```rider```: a row in the rider feature DataFrame
    - label riders by the their riding frequency specified in the 'total_num_trips' column
      - 0: infrequent riders riders who ride less than 5 times per month
      - 1: riders who ride less than or equal to 20 times per month
      - 2: riders who ride more than 20 times per month
      - -1: others
    - return label

  - ```_label_commuter_rail_rider(self, rider)```:
    - argument ```rider```: a row in the rider features DataFrame
    - label commuter rail riders except for those coming from zone 1A as 'CR except zone 1A' and 'others' otherwise
    - return label

  - ```extract_features(self)```:
    - call ```_extract_temporal_patterns()```, ```_extract_geographical_patterns()```, ```_extract_ticket_purchasing_patterns()``` to extract each group of features
    - drop infrequent riders and commuter rail riders from zones other than 1A
    - column-wise concatenate features to and save as a `.csv` file to the features cache path
    - return the concatenated rider features DataFrame

## Rider Segmentation

### Class ```Segmentation```

A ```Segmentation``` object clusters riders by their temporal, geographical, and ticket-purchasing features based on a user-specified pipeline option (hierarchical vs. non-hierarchical), a user-specified algorithm option (kmeans vs. lda) and a user-specified feature weighs.

- **Attributes**:
  - ```random_state```, ```max_iter```, ```tol```: attributes for initializing K-means
  - ```start_month```: a string representing the start month in the format of YYMM, e.g. '1710'
  - ```duration```: an integer representing the length of duration
  - ```w_time_choice```: an integer from 0 to 100 representing the weight of temporal features as percentage
  - ```N_riders```: number of riders in the rider features DataFrame
  - ```time_feats```: a list of column name of all sets of temporal features
  - ```geo_feats```: a list of column names of all geographical features
  - ```purchase_feats```: a list of column names of all ticket-purchasing features
  self.weekday_vs_weekend_feats = ['weekday', 'weekend']
  - ```features```: a list of concatenated time, geo and ticket-purchasing features for non-hierarchical clustering
  - ```features_layer_1```: a list of features used in the initial clustering step
  - ```features_layer_2```: a list of features used in the final clustering step
  - ```w_time```: weight for temporal patterns
  - ```w_geo```: weight for geographical patterns
  - ```w_purchase```: weight for purchase patterns
  - ```w_week```: weight for weekend/weekday columns
  - ```df```: the rider features DataFrame

- **Methods**:
  - ```__init__(self, w_time=None, start_month='1701', duration=1, random_state=RANDOM_STATE, max_iter=MAX_ITER, tol=TOL)```:
    - initialize the attributes
    - call ```__get_data()```, ```__standardize_features()```, ```__normalize_features()```

  - ```__get_data(self)```:
    - read in and save as ```self.df``` the extracted rider features DataFrame of the specified `start_month` and `duration`

  - ```__standardize_features(self)```:
    - standardize numeric features so that each column has mean = 0 and std = 1
    - a column with the same value (std = 0) would be set to 0's

  - ```__normalize_features(self)```:
    - normalize numeric features so that each column has max = 1 and min = 0

  - ```__apply_clustering_algorithm(self, features, model, n_clusters_list=[2, 3, 4, 5])```:
    - argument `features`: the rider features DataFrame
    - argument `model`: an initialized but not fitted sklearn K-means or LDA object
    - argument `n_clusters_list`: a list of integers to representing the range of possible number of clusters
    - segment the `features` using the segmentation `model` for all values in `n_clusters_list`
    - for each value used as the number of clusters, the clustering assignment is evaluated by calling ```__get_cluster_score(self, features, cluster_labels)``` to get a Calinski-Harabaz Index score
    - return the list of cluster assignments with the highest Calinski-Harabaz Index score

  - ```__get_cluster_score(self, features, cluster_labels)```:
    - argument `features`: rider features DataFrame
    - argument `cluter_labels`: a list of integers representing cluster assignments
    - return the Calinski-Harabaz Index score that shows how good the clustering results are

  - ```__initial_rider_segmentation(self, hierarchical=False)```
    - argument `hierarchical`: a boolean indicator of whether the segmentation pipeline is hierarchical or not
    - With a **hierarchical** pipeline:
      - perform initial clustering using K-means algorithm on the [weekday-vs-weekend trip count features](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#feature-set-2-weekday-vs-weekend-total-counts) and [ticket-purchasing features](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#ticket-purchasing-pattern)
      - append the initial cluster assignments to the rider features DataFrame as a new column 'initial_cluster'
    - With a **non-hierarchical** pipeline:
      - simply rename the column 'group_by_frequency' to 'initial_cluster'

  - ```__final_rider_segmentation(self, model, features, n_clusters_list=[2, 3, 4, 5], hierarchical=False):```:
    - argument `hierarchical`: a boolean indicator of whether the segmentation pipeline is hierarchical or not
    - set the feature weights
    - With a **hierarchical** pipeline:
      - within each initial cluster, perform further clustering on [168 hourly temporal features](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#feature-set-1a-168-hourly-version) and [geographical features](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#geographical-patterns)
    - With a **non-hierarchical** pipeline:
      - perform clustering on [168 hourly temporal features](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#feature-set-1a-168-hourly-version), [geographical features](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#geographical-patterns) and [ticket-purchasing features](https://ac297r-mbta-2018.github.io/Final-Report/feature.html#ticket-purchasing-pattern)
    - save the final cluster assignments to the rider features DataFrame as a new column 'final_cluster'
    - return the 'final_cluster' column

  - ```get_rider_segmentation(self, hierarchical=False)```:
    - argument `hierarchical`: a boolean indicator of whether the segmentation pipeline is hierarchical or not
    - apply the hierarchical or non-hierarchical pipeline to get the final cluster assignments
    - save the results as `.csv` files to the cluster cache path

## Cluster Inference

### Class ```CensusFormatter```

A ```CensusFormatter``` object formats the census data to counts, percentages or proportions based on the user's specification.

- **Attributes**:
  - `new_col_names`: static class variable, a list of column names used to rename raw census columns
  - `census_groups`: static class variable, a dictionary for census groups and prefixes in the renamed census DataFrame
  - `raw_census_filepath`: a string representing the file path of the raw census data
  - `census_in_counts`: a DataFrame of census data represented in counts
  - `census_in_percents`: a DataFrame of census data represented in percentages
  - `census_in_proportions`: A DataFrame of census data represented in proportions

- **Methods**:
  - `__init__(self, raw_census_filepath)`:
    - initialize the attributes
    - format the raw census data in counts, percentages and proportions, respectively

  - `__format_raw_census_in_counts(self, raw_census_filepath)`:
    - argument `raw_census_filepath`: a string of file path to raw census data
    - format the raw census data as counts

  - `__convert_to_percents(self, census_in_counts)`:
    - argument `census_in_counts`: a DataFrame of census data represented in counts (the output of `__format_raw_census_in_counts()`)
    - format the raw census data as percentages

  - `__convert_to_proportions(self, census_in_counts)`:
    - argument `census_in_counts`: a DataFrame of census data represented in counts (the output of `__format_raw_census_in_counts()`)
    - format the raw census data as proportions

  - `to_csv(self, filename, census_type='proportions')`:
    - argument `filename`: A string of file name to save
    - argument `census_type`: A string of which census type to save, options = ['percents', 'counts', 'proportions'], default = 'proportions'
    - save census data in the type specified by `census_type`

  - `get_census_in_counts(self)`:
    - return census_in_counts

  - `get_census_in_percents(self)`:
    - return `census_in_percents`

  - `get_census_in_proportions(self)`:
    - return `census_in_proportions`

### Class `ClusterProfiler`:

A `ClusterProfiler` object summarizes each cluster's overall pattern-of-use features and infers its demographics distributions based on the mapping from its softmax transformed geographical patterns to the census data.

- **Attributes**:
  - `feat_groups`: static class variable, a list of feature groups and expected prefixes in rider features DataFrame
  - `demo_groups`: static class variable, a dictionary for demographics groups and prefixes in the profiled cluster DataFrame
  - `start_month`: a string representing the start month in the format of YYMM, e.g. '1710'
  - `duration`: an integer representing the length of duration
  - `hierarchical`: a boolean indicator of whether the segmentation pipeline is hierarchical or not
  - `census`: a DataFrame of census data represented in counts returned by a `CensusFormatter` object
  - `w_time`: an integer from 0 to 100 representing the weight of temporal features as percentage
  - `input_path`: a string of the cached cluster results
  - `param_keys`: a list of parameter keys
  - `riders`: the rider feature DataFrame with cluster assignments

- **Methods**:
  - `__init__(self, hierarchical=False, w_time=None, start_month='1701', duration=1)`:
    - initialize the attributes
    - call `__get_data()` to get the rider features DataFrame with cluster assignments

  - `__split(self, delimiters, string, maxsplit=0)`:
    - split file name by `delimiters` to match cached results

  - `__get_cached_params_list(self)`:
    - find the parameter combinations of the cached results
    - return the list of parameter combinations of all cached results

  - `__get_data(self)`:
    - read in the cached cluster results or re-cluster to get the rider features DataFrame with cluster assignments
    - save the rider features DataFrame with cluster assignments in the `self.riders` attribute

  - `_softmax(self, df)`:
    - argument `df`: a DataFrame of numeric values
    - transform each row of `df` into softmax probabilities
    - return the transformed DataFrame

  - `_summarize_features(self, riders, by_cluster)`:
    - argument `riders`: a DataFrame containing rider-level pattern-of-use features used to form clusters plus resulting cluster assignment
    - argument `by_cluster`: a boolean indicating whether to summarize features by cluster or overall
    - summarize, by cluster or overall, rider-level pattern-of-use features by turning each feature group (e.g. temporal or geographical patterns) into a probability distribution (columns belonging to a feature set add up to 1)

  - `_summarize_demographics(self, cluster_features)`:
    - argument `cluster_features`: a DataFrame containing cluster-level pattern-of-use features
    - take a softmax transformation on the geo pattern distribution of each cluster
    - compute the expected demographics distribution using the softmax distribution of geo patterns
      - In other words, we calculate E = $\sum_i x_i p(x_i)$ where x is a vector representing demographics data in zip code i and $p(x_i)$ is the probability of zipcode i.

  - `_get_first_2_pca_components(self, features)`:
    - argument `features`: a DataFrame containing cluster-level pattern-of-use features
    - compute the first 2 principal components (PCs) of the cluster-level pattern-of-use features
    - save the 2 PCs' values and the cluster size as a DataFrame
    - return the DataFrame

  - `extract_profile(self, algorithm, by_cluster)`:
    - argument `algorithm`: a string, options = ['kmeans', 'lda']
    - argument `by_cluster`: a boolean indicating whether to summarize features by cluster or overall
    - summarize the cluster pattern-of-use features, demographics distributions and the first 2 PCs
    - save the profiled cluster summaries by calling `__save_profile(self, profile, algorithm, by_cluster)`

  - `__save_profile(self, profile, algorithm, by_cluster)`:
    - argument `profile`: a DataFrame of summarized cluster pattern-of-use features with its demographics distribution
    - argument `algorithm`: a string, options = ['kmeans', 'lda']
    - argument `by_cluster`: a boolean indicating whether to summarize features by cluster or overall
    - format the profile cache path and save the profiled cluster summaries in the path

## Visualization

### Class `Visualization`:
A `Visualization` object visualizes the cluster profiles in various types of visualizations (i.e. static heatmap for cluster temporal patterns, static scatter chart for visualizing clusters on 2D PCA-subspace, interactive map for cluster geographical patterns, and static bar charts for other cluster statistics)

- **Attributes**:
  - `start_month`: a string representing the start month in the format of YYMM, e.g. '1710'
  - `duration`: an integer representing the length of duration
  - `input_path`: a string for the input directory path (path to cached profiles)
  - `output_path`: a string for the output directory path
  - `param_keys`: a list of parameter keys for matching user-specified options to the cached cluster profiles (this is primarily for reading in the data)
  - `df`: the DataFrame with cluster profiles data for visualization
  - `req_view`: User-specified view request, options are ["overview", "hierarchical", "non-hierarchical"]. "Overview" is the option to view the overall pattern where all riders are treated as one big cluster. "Hierarchical" and "non-hierarchical" are options to view the clustering results from the hierarchical or the non-hierarchical pipeline.
  - `req_w_time`: User-specified weights on temporal patterns, possible values are integers from 0 to 100. Note, equal weighting between temporal, geographical, and ticket purchasing patterns is assumed if this weight is set to 0 or is left unspecified.
  - `req_algo`: User-specified algorithm request, options are ["lda", "kmeans"] for viewing the clustering results from LDA or K-means algorithms, repspectively.

- **Methods**:
  - `__init__(self, start_month='1701', duration=1)`:
      - initialize the some class attributes (i.e. `start_month`, `duration`, `input_path`, `output_path` and `param_keys`)
      - set up the output directory based on `output_path` if it does not already exist

  - `__split(self, delimiters, string, maxsplit=0)`:
    - split file name by `delimiters` to match cached results

  - `__get_cached_params_list(self)`:
    - find the parameter combinations in the input path directory
    - return the list of parameter combinations of all cached results

  - `__read_csv(self, req_param_dict, by_cluster)`:
    - read in the cached cluster profile results based on user-specified requests (req_param_dict, which includes requests specified for view option, start month, duration, weight on temporal patterns and algorithm). The by_cluster paramter is a boolean, where True means viewing by cluster and False means viewing the overall pattern
    - save the read DataFrame in the `self.df` attribute

  - `load_data(self, by_cluster=False, hierarchical=False, w_time=None, algorithm=None)`:
    - parse user-specified parameters to construct a dictionary of requested parameters (req_param_dict) to either call the `__read_csv()` function for reading in the data if the requested file exists in the cached_profile directory or make a `ClusterProfiler` to extract the requested cluster profile

  - `visualize_clusters_2d(self)`:
    - plot the clusters in the 2D PCA subspace on a static scatter plot. This allows a visual comparison for how different the clusters are from each other.

  - `plot_cluster_hourly_pattern(self, cluster)`:
    - given a cluster ID, plot a 7 (days of week) by 24 (hours in a day) temporal usage matrix using a heatmap visualization

  - `plot_all_hourly_patterns(self)`:
    - plot the temporal usage heatmap visualizations for all clusters in the `self.df` attribute

  - `plot_cluster_geo_pattern(self, cluster)`:
    - given a cluster ID, plot an interactive map visualization for visualizing the cluster geographical pattern.
    - save the resulting visualization as a html file in the output_path directory (typically `cached_viz` unless the user resets the path in `config.py`)

  - `__single_feature_viz(self, feature, title, ylabel, xlabel)`:
    - helper method for `plot_cluster_size` and `plot_avg_num_trips` functions below to plot a single bar chart of cluster feature VS. cluster ID

  - `__group_feature_viz(self, grp_key, stacked, title, ylabel, xlabel)`:
    - helper method for `plot_demographics` and `plot_ticket_purchasing_patterns` functions below to plot Multi-bar plot visualizations of cluster feature VS. cluster ID

  - `plot_cluster_size(self)`:
    - plot number of riders vs. cluster ID on a static bar chart

  - `plot_avg_num_trips(self)`:
    - plot average number of trips vs. cluster ID on a static bar chart


  - `plot_demographics(self, grp, stacked=True)`:
    - plot either a "stacked" or "grouped" barchart showing the inferred cluster demographics distributions
    - the `grp` option specifies which type of demographics distribution to display. Options are ['race', 'emp', 'edu', 'inc'] for race, employment, education and income.

  - `plot_ticket_purchasing_patterns(self, grp, stacked=True)`:
    - plot either a "stacked" or "grouped" bar chart showing cluster ticket purchasing patterns
    - the `grp` option specifies which type of ticket purchasing habit to display. Options are ['servicebrand', 'usertype', 'tariff'] for service brand (e.g. Rapid Transit), user type (e.g. Adult or Student), and tariff type (e.g. Monthly Pass).

## Auto Report Generator

### Class `ReportGenerator`

A `ReportGenerator` object is initialized in a `ClusterProfiler` object. It generates a text summary for each cluster based on the output of the `ClusterProfiler` that contains it and a pre-trained (and retrainable) Convolutional Neural Network (CNN) model for the 7x24 temporal pattern classification.

- **Attributes**:
  - `n_classes`: an integer indicating the number of different types of riders to classify
  - `cnn_model_filename`: a string of the file name (`.h5`) of the pre-trained CNN model
  - `sample_factor`: an integer indicating the factor for oversampling clusters' 7 x 24 time matrices
  - `noise_std`: a float indicating the standard deviation for the Gaussian noise signal used in oversampling the time matrices

- **Methods**:
  - `get_text(self, row)`:
    - argument `row`: a row of the profiled cluster DataFrame
    - return the formatted string of generated text description for each cluster, including the predicted rider-type, the cluster size, the average number of trips, the most frequent trip hours during weekdays / weekends and the zip code that shows most traffic

  - `generate_report(self, df)`:
    - argument `df`: a DataFrame of the profiled cluster
    - predict the rider-type for each row in `df` using the pre-trained CNN model
    - generate text summary for each cluster
    - append the text summaries as new column ('report') to `df`
