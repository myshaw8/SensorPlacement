# Water Distribution Network Sensor Placement
<!-- TABLE OF CONTENTS -->
   1. [About the Project](#about-the-project)
   2. [Built With](#built-with)
   3. [Getting Started](#getting-started)
   4. [Report](#report)
      1. [IP Programming](#ip-programming)
      2. [Maximizing Network Coverage](#maximizing-network-coverage)
      3. [Comparison: Greedy Algorithm](#comparison-greedy-algorithm)
      4. [Minimizing Criticality of Unmonitored Pipes](#minimizing-criticality-of-unmonitored-pipes)
      5. [Minimizing Criticality: Improved Placement](#minimizing-criticality-improved-placement)
   5. [Acknowledgements](#acknowledgements)

<!-- ABOUT THE PROJECT -->
## About the Project
Water distributions networks comprise a portion of critical infrastructure that is vital for our modern daily life. Disruptions to this infrastructure can result in catastrophic consequences for societal functions. As such networks are both complex, large, and geographically distributed, proactive monitoring is crucial for detecting failures quickly and accurately.

This project uses a modified version of the [KY2 Network](https://uknowledge.uky.edu/wdst/4/) for analysis. The network comprises 1123 pipes and 811 nodes.

Information about the network is contained in two files:
**Detection_Matrix.csv** | A binary matrix showing the the detection ability of a sensor placed at a given node to detect a failure in a given pipe.
**Criticality.csv** | A table showing a criticality factor (0-1) of a pipe. Greater criticality implies greater importance.

The objective of this project is to determine the placement of sensors under different conditions. It will evaluate:
1. Optimal sensor placement to maximize network coverage
2. Comparison of a greedy sensor placement algorithm
3. Sensor placement, maximizing covered pipes' criticality
4. Sensor placement, factoring in uncovered pipes' criticality

The KY2 Water Distribution Network is shown below. Yellow dots represent detection nodes, blue links represent pipes.

<img src="images/network.png?raw=true" width="700"/>

<!-- BUILT WITH -->
## Built With
* Python - Pandas, Seaborn, PuLP
* Gurobi Optimizer

<!-- GETTING STARTED -->
## Getting Started
The analysis for this project was done in a jupyter notebook. [Repository](https://github.com/myshaw8/SensorPlacement).

Running Python 3.9.5 and the requisite packages should suffice. Gurobi optimizer usually requires an commercial license, but use was obtained using a temporary academic license. You may need to substitute the optimizer for other open-source ones.

## Report
### IP Programming
Given the fixed detection matrix and the fact that placement is a "yes/no" variable, the optimization problem results in a binary integer program.
Each pipe and node must have its own binary variable, tracking detection and placement respectively.

```python
#Binary variables denoting if a sensor is placed on a node
node_varsA = LpVariable.dicts("",pd_DM.columns,0,1,cat='Binary')

#Binary variables denoting if a sensor is detecting a pipe
pipe_varsA = LpVariable.dicts("",pd_DM.index,0,1,cat='Binary')
```

Furthermore, there must be linkages betwen the pipe and node variables:
1. If a node has a sensor, all associated pipes have detection
2. A pipe does not have detection if and only if all associated nodes lack a sensor

```python
#Linking constraints: pipe binary var >= associated node binary var
for n in nodelist:
    for p in pd_DM[n][pd_DM[n] == 1].index: #find all pipes linked to a node
        problemA += pipe_varsA[p] >= node_varsA[n] # setup a constraint for each pipe-node combination

#Linking constraints: making sure that p is zero if none of the nodes have sensors
for p in pipelist: #for each pipe
    all_rel_nodes = pd_DM.loc[p][pd_DM.loc[p] == 1].index #find all nodes touching a pipe
    problemA += pipe_varsA[p] <= lpSum([node_varsA[n] for n in all_rel_nodes]) #ensure the constraint
```

The objective function will vary for multiple scenarios.

### Maximizing Network Coverage
The objective function maximizes the number of pipes detected.

```python
#Objective function - sum of pipes that are detected by a sensor
problemA += lpSum([pipe for pid, pipe in pipe_varsA.items()])
```

By varying the number of placed sensors from 0-20, we see a logarithmic increase in the number of detected pipes.

<img src="images/coverage.png?raw=true"/>

### Comparison: Greedy Algorithm
The objective function maximizes the number of pipes detected, but in a greedy manner (selected one at a time, until all pipes are selected).
While this is performed using a mathematical program, it can also be done more easily by filtering columns/rows from a pandas dataframe.

The greedy algorithm has slightly inferior performance to a global optimization program, but demonstrates the potential of a heuristic approach.

<img src="images/greedy.png?raw=true"/>

### Minimizing Criticality of Unmonitored Pipes
Minimizing the criticality of an unmonitored pipe is a min-max problem. Auxiliary variable Z is minimized and is set greater than or equal to the criticality of all unmonitored pipes.

```python
#Min Max problem: Min Z
problemE = LpProblem("Sensor_Placement", LpMinimize)
problemE += auxz #that's the objective

#Subject to z + yiwi >= (1 - xi) * wi where xi is a detected pipe and wi is its criticality
# z >= (1 - xi) * wi - yiwi
for pid, pipe in pipe_varsE.items():
    problemE += auxz >= (1 - pipe) * pd_crit_int[pid]
```

Varying placed sensors from 0-20 results in a exponential decrease in unmonitored criticality.

<img src="images/criticality-1.png?raw=true"/>

For 1-2 sensors placed, there are multiple optimal solutions, resulting in maximum unmonitored criticality of 1.0 and 0.7 respectively.
Since there are no additional constraints, the solver selects nodes that have very few pipes that can be monitored.

<img src="images/criticality-2.png?raw=true"/>

### Minimizing Criticality: Improved Placement
In order to improve the program, a penalty term is introducted to the objective function, representing the average unmonitored criticality of all nodes.

```python
#Min Max problem: Min Z
problemEU = LpProblem("Sensor_Placement", LpMinimize)
problemEU += auxz - lpSum([pipe * pd_crit_int[pid]/100 for pid, pipe in pipe_varsEU.items()]) / 1123  #that's the objective
```

Unmonitored criticality performs similarly to the original solution when varying number of sensors.

<img src="images/criticality-improved-1.png?raw=true"/>

The pipe coverage is now logarithmic for the number of sensors placed. For the previous case of 1-2 sensors with multiple optimal solutions, the sensors are now placed at nodes that maximize the covered criticality of the sensor network. This results in more reasonable results for sensor placement.

<img src="images/criticality-improved-2.png?raw=true"/>

## Acknowledgements
The Sensor Placement project was the course project for ISYE 6333 - Operations Research for Supply Chains, taught by [Dr. Mathieu Dahan](https://www.isye.gatech.edu/users/mathieu-dahan) at Georgia Tech in Fall 2021.

Thanks to my teammates [Aurimas Racas](https://www.linkedin.com/in/aurimas/) and [Hayden Dessomes](https://www.linkedin.com/in/haydendessommes/) for their contributions to the project.
