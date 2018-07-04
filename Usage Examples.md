---
nav_include: 2
title: Usage & Examples
---

## Contents
{:.no_toc}
*  
{: toc}


## Feature Extraction

In the same directory level as MBTAriderSegmentation, run the following code. The extracted features would be saved to ```MBTAriderSegmentation/data/cached_features``` directory by default. The default paths can be changed in ```MBTAriderSegmentation/config.py```

```python
import time
from MBTAriderSegmentation.config import *
from MBTAriderSegmentation.features import FeatureExtractor

import time
t0 = time.time()
start_month = '1710' # specify start month
duration = 1 # specify length of duration
extractor = FeatureExtractor(start_month=start_month, duration=duration).extract_features()
print("feature extraction time: ", time.time() - t0)
```
## Rider Segmentation

In the same directory level as MBTAriderSegmentation, run the following code. The clustering results would be saved to ```MBTAriderSegmentation/data/cached_clusters``` directory by default. The default paths can be changed in ```MBTAriderSegmentation/config.py```

```python
import time
from MBTAriderSegmentation.config import *
from MBTAriderSegmentation.segmentation import Segmentation

# Hierarchical pipeline
import time
t0 = time.time()
start_month = '1710'
duration = 1
segmentation = Segmentation(start_month=start_month, duration=duration)
segmentation.get_rider_segmentation(hierarchical=True)
print("Hierarchical clustering time: ", time.time() - t0)

# Non-hierarchical pipeline
t0 = time.time()
start_month = '1710' # specify start month
duration = 1 # specify length of duration
segmentation = Segmentation(start_month=start_month, duration=duration)
segmentation.get_rider_segmentation(hierarchical=False)
print("Non-hierarchical clustering time: ", time.time() - t0)
```
## Cluster Inference

In the same directory level as MBTAriderSegmentation, run the following code. The profiled cluster summary files would be saved to ```MBTAriderSegmentation/data/cached_profiles``` directory by default. The default paths can be changed in ```MBTAriderSegmentation/config.py```

```python
import time
from MBTAriderSegmentation.config import *
from MBTAriderSegmentation.profile import ClusterProfiler

# by_cluster = True
start_month = '1710' # specify start month
duration = 1 # specify length of duration
hier_flags = [True, False]
algo_types = ['kmeans', 'lda']
# by_cluster = True
for hier_flag in hier_flags:
    for algo_type in algo_types:
        t0 = time.time()
        profiler = ClusterProfiler(start_month=start_month, duration=duration, hierarchical=hier_flag)
        profiler.extract_profile(algorithm=algo_type, by_cluster=True)
        print("[by_cluster = True] Profile time: ", time.time() - t0)

# by_cluster = False
t0 = time.time()
profiler = ClusterProfiler(start_month=start_month, duration=duration, hierarchical=True)
profiler.extract_profile(algorithm='kmeans', by_cluster=False)
print("[by_cluster = False] Profile time: ", time.time() - t0)

```

## Visualization

In the same directory level as MBTAriderSegmentation, run the following code. The visualizations for temporal patterns, cluster pattern-of-use and demographics plots would pop up and the map visualization for the geographical features would be saved as html files to ```MBTAriderSegmentation/data/cached_viz``` directory by default. The default paths can be changed in ```MBTAriderSegmentation/config.py```

```python
import time
from MBTAriderSegmentation.config import *
from MBTAriderSegmentation.visualization import Visualization

# load data
start_month = '1710' # specify start month
duration = 1 # specify length of duration
viz = Visualization(start_month=start_month, duration=duration)
viz.load_data(by_cluster=True, hierarchical=True, w_time=None, algorithm='lda')

# plot cluster_size, avg_num_trips
viz.plot_cluster_size()
viz.plot_avg_num_trips()

# PCA plot
viz.visualize_clusters_2d()

# plot temporal patterns
viz.plot_all_hourly_patterns()

# generate geographical patterns as html files
unique_clusters = list(viz.df['cluster'].unique())
map_urls = []
for cluster in unique_clusters:
    map_urls.append(viz.plot_cluster_geo_pattern(cluster))

# plot demographics distribution
viz.plot_demographics(grp='race', stacked=True)
viz.plot_demographics(grp='edu', stacked=False)

```
