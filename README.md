# Overview

Example integration of [SLEAP](https://sleap.ai/) and [Motile](https://github.com/funkelab/motile) for multi object tracking with integer linear equations. The hope is to extend this with new costs/constraints that are also applicable for pose estimation. Ideally, Motile will be useful as a standalone package for multi-object tracking, and agnostic to the tracking task (e.g should be able to work for both cell lineage tracking and animal tracking). 

# Installation Instructions

1) download example data:

```
wget https://www.dropbox.com/s/6gnntxvs2s17rmm/test_data.zip && unzip test_data.zip
```

2) create conda env:

```
conda create -n sleap_motile -c funkey pylp
conda activate sleap_motile
```

3) install motile. Note: this example was tested with an older version of motile which has since been updated. For now just use old version:

```
git clone https://github.com/funkelab/motile.git
cd motile
git checkout 824278f
pip install .
cd ../
```

4) install sleap (pip is fine for this example):

```
pip install sleap==1.2.9
```

5) run example (`example_use.ipynb`) in jupyter notebook or jupyter lab (might need to install first), eg:

```
pip install jupyterlab
jupyter-lab
```

# This example

## Very simple case

```
* 2 trajectories (e.g flies) over 3 frames
* Just consider x dim for now. Take euclidean distance as cost
* Edge between id 0 (x = 101) and id 2 (x = 100) -> 1.0 (should be selected)
* Edge between id 1 (x = 150) and id 2 (c = 100) -> 50.0 (should not be selected)

   x            
   |         
150|   1---3---5
   |     x   x
100|   0---2---4
    --------------- t
       0   1   2
  
* Solver should select edges: [(0, 2), (1, 3), (2, 4), (3, 5)]  
```

## Added node (max_children=1)

```
* Same example but now we have an extra node (6)

   x            
   |
200|           6
   |         /
150|   1---3---5
   |     x   x
100|   0---2---4
    ---------------- t
       0   1   2

* Solver should select edges: [(0, 2), (1, 3), (2, 4), (3, 5)]   
```

## Added node (max_children=2)

```
* Same example but now we have an extra node (6)

   x            
   |
200|           6
   |         /
150|   1---3---5
   |     x   x
100|   0---2---4
    ---------------- t
       0   1   2

* Solver should select edges: [(0, 2), (1, 3), (2, 4), (3, 5), (3, 6)]
```

## Integrating with SLEAP

The `test_data` includes an example SLEAP file and movie. The movie consists of two flies in an arena, and the .slp file includes the ground truth [labels](https://sleap.ai/api/sleap.io.dataset.html?highlight=labels#sleap.io.dataset.Labels) (instances with poses defined by nodes and edges). The first three frames look like this:

![](https://github.com/sheridana/sleap_motile/blob/main/flies.png)

In the example, the `get_nodes` function will parse the anchor nodes in the sleap file for a given amount of frames and return a list of dicts in the same format to the prior examples. E.g `nodes = get_nodes(labels, num_frames=3)` will return: 

| id | t | x          | y          | score |
|----|---|------------|------------|-------|
| 0  | 0 | 448.760010 | 258.080017 | 1.0   |
| 1  | 0 | 536.160034 | 239.240005 | 1.0   |
| 2  | 1 | 448.919983 | 258.080017 | 1.0   |
| 3  | 1 | 535.839966 | 239.160004 | 1.0   |
| 4  | 2 | 448.919983 | 258.000000 | 1.0   |
| 5  | 2 | 535.239990 | 238.759995 | 1.0   |

We then just take the euclidean distance between nodes across consecutive frames, forward in time to get our edges and prediction_distances. E.g `get_edges(nodes)` will then give us this:

| source | target | prediction_distance |
|:------:|:------:|:-------------------:|
|    0   |    2   |       0.025591      |
|    0   |    3   |     7940.885655     |
|    1   |    2   |     7965.772582     |
|    1   |    3   |       0.108844      |
|    2   |    4   |       0.006403      |
|    2   |    5   |     7824.406937     |
|    3   |    4   |     7910.028891     |
|    3   |    5   |       0.519978      |

Passing these nodes and edges to motile as before (`test_solver_motile(nodes, edges)`) should then select the same edges:

`Selected edges: [(0, 2), (1, 3), (2, 4), (3, 5)]`

Selecting nodes over more frames should work the same.

Todos:

* Add as example to motile docs when ready
* Test integration with transformer association matrix
* Extend to poses
* Add more animal / pose specific costs and constraints

---

# Updated example

* Added an updated example (`example_use_updated.ipynb`) which uses the most recent changes to `motile`.
* So installing motile can simple be changed to:

```
pip install git+https://github.com/funkelab/motile.git
```

* Changes `prediction_distance` to just `distance` since there is no predicted movement in this example
* Replaces `EdgeSelection` cost with `EdgeDistance` cost
* Adds `Split` cost
* Now simply setting `max_children=2` won't select node 6 unless the split_cost is also lowered
