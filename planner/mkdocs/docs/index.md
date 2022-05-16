# FI-planners

More details of FI-planners are explained in this paper, [Reshaping Diverse Planning](http://www.cs.toronto.edu/~shirin/AAAI-KatzM.5922.pdf). The planners can be accessed [here](https://github.com/IBM/forbiditerative).

## Top k planner

The top k planner returns a number of k best quality plans. We used this planner in the AAMAS paper. The plans we got are plans with the lowest costs (e.g. if k = 100, then we will get the first 100 best plans starting from a lowest cost plan).

```sh
# ./plan_topk.sh <domain> <problem> <number-of-plans>
./plan_topk.sh examples/logistics00/domain.pddl examples/logistics00/probLOGISTICS-4-0.pddl 100
```

## Top quality planner

To use the top quality planner, we need set a quality-multiplier. For example, if quality-multiplier = 1.1, then this planner will return all the plans with a cost less than the lowest cost $\times$1.1, the lowest cost is the cost of an optimal plan.

```sh
# ./plan_topq_via_topk.sh <domain> <problem> <quality-multiplier>
./plan_topq_via_topk.sh examples/logistics00/domain.pddl examples/logistics00/probLOGISTICS-4-0.pddl 1.1
```

## Diverse planner

There are 3 available diverse planners, **diverse_agl**, **diverse_sat** and **diverse_bD**.

- #### Diverse_agl

The diverse_agl planner is the foundation of the diverse_sat planner and the diverse_bD planner. The diverse_agl planner is similar to the top k planner mentioned above, it will take a number-of-plan (k) as input and then will return k plans from the lowest cost plan to the plans with higher costs (k best plans). However, the difference between these two planner is whether the planner will distinguish two plans with same actions but different orders. For example, plan 1 has an action sequence <A, A, B, D, C>, plan 2 has an action sequence <B, D, A, A, C>. In the top k planner, these are **two** different plans, but the diverse_agl planner treats these as **one** plans. For the diverse_agl planner, different plans must have different sets of actions.

```sh
# ./plan_diverse_agl.sh <domain> <problem> <number-of-plans>
./plan_diverse_agl.sh examples/logistics00/domain.pddl examples/logistics00/probLOGISTICS-4-0.pddl 10
```

- #### Diverse_sat

The diverse_sat planner build on top of the diverse_agl plan. These two steps. The first step is the diverse_agl planner generating M number of plans. Then, the second step is selecting N number of plans from the plans generated by step 1. Notice that M need to be larger than N.

For the selection stage, we need to rely on some diversity metrics for filtering the valid plans. There are 3 metrics provided (currently available) which are stability, uniqueness, state. Once a metric be chosen, the diverse_sat planner will get N valid plans out of M plan candidates generated in step 1. The selecting step follows a greedy algorithm, given a currently selected plan set, it aims to add a next plan that can maximize the metric for the current selected plan set. So the output not necessarily to be the optimal N valid plans.

```powershell
## See the dependencies below (1 and 2)
# ./plan_diverse_sat.sh <domain> <problem> <number-of-plans> <diversity-metric> <larger-number-of-plans>
./plan_diverse_sat.sh examples/logistics00/domain.pddl examples/logistics00/probLOGISTICS-4-0.pddl 10 stability 20
```

For the above code, the diverse_agl planner generates 20 plans. Then, from these 20 plans, select 10 plans which can maximizing the metric (greedily).

- #### Diverse_bD

The diverse_bD planner is similar as the diverse_sat, it post-process the founded plans from diverse_agl according to one of the metrics (stability, uniqueness, state). But in the selection stage, this planner set a boundary of the metric, if a next plan can satisfy the metric boundary requirement, then it will be add into the output set.

```sh
## See the dependencies below (1, 2, and 3)
# ./plan_diverse_bounded.sh <domain> <problem> <number-of-plans> <diversity-metric> <bound> <larger-number-of-plans>
./plan_diverse_bounded.sh examples/logistics00/domain.pddl examples/logistics00/probLOGISTICS-4-0.pddl 10 stability 0.25 20
```

### Diversity metrics

Both the diverse_sat planner and the diverse_bD planner need a ``diversity-metric`` as input. Generally, ``diversity-metric`` is the **average number of pairwise dissimilarity** (pairwise plan distance), $\delta(\pi,\pi')=1-sim(\pi,\pi')$. And $sim(\pi,\pi')$ is the pairwise similarity between two plans $\pi$ and $\pi'$, it is a number between 0 (two plans are unrelated) and 1 (two plans are equivalent). There are 3 ways (metrics) to compute the $sim(\pi,\pi')$ as below.

**1. Stability**

$$
sim_{stability}(\pi,\pi')=\frac{|A(\pi)\cap A(\pi')|}{|A(\pi)\cup A(\pi')|}
$$
It is a fraction of the number of actions occur in both plans over the total number of actions of these two plans. Notice that both A($\pi$) and A($\pi'$) stand for action sets. Related works are [Fox et al.2006](https://www.aaai.org/Papers/ICAPS/2006/ICAPS06-022.pdf); [Coman and Munoz-Avila 2011](http://www.cse.lehigh.edu/~munoz/Publications/aaai11.pdf).

**2. Uniqueness**

$$sim_{uniqueness}(\pi,\pi') =
\begin{cases}
0, & \text{if $A(\pi)$ \ $A(\pi')=\emptyset$} \\
0, & \text{if $A(\pi') \subset A(\pi)$} \\
1, & \text{otherwise}
\end{cases}$$

It only considers plans as action sets, A($\pi$) and A($\pi'$) are sets of $\pi$ and $\pi'$. $A(\pi) \mbox{ \\ } A(\pi')$ means whether $A(\pi)$ has unique actions that not in $A(\pi')$. Therefore, the uniqueness of two plans is 1 if both plans have unique actions not occur in the other plan, otherwise uniqueness is 0 ([Roberts, Howe, and Ray 2014](http://makro.ink/publications/robertsHoweRay14.icaps.evaluating.pdf)).

**3. State**

Let $\pi$ and $\pi'$ be two plans, $\{s_0,s_1,...,s_m\}$ and $\{s'_0,s'_1,...,s'_n\}$ are the sequence of states if we apply $\pi$ and $\pi'$. States are consist of predicates and let m $\leq$ n. Notice that from the states m+1 to n are ignored in this paper.
$$
sim_{state}(\pi,\pi')=\frac{1}{m} \times \sum_{i=1}^{m}{\Delta(s_i,s'_i)}
$$

$$
\Delta(s_i,s'_i)=1-\frac{|s_i \cap s'_i|}{|s_i \cup s'_i|}
$$

For example, if $s_0=\{r_0,r_1\}$, $s'_0=\{r_0,r_2\}$, then $\Delta(s_0,s'_0)=\frac{2}{3}$. The related work can be found here ([Nguyen et al. 2012](https://www.sciencedirect.com/science/article/pii/S0004370212000707)).
