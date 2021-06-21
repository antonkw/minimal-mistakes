---
title: "Solve CNF with Tarjan"
categories:
  - Algorythms
tags:
  - algorythms
  - scala
  - immutability
  - collection
  - performance
---

Today I want to describe my journey of writing a purely functional implementation for solving 2-satisfiability problem.
Essentially it is a problem of assignment of boolean values to the given boolean formula. It is the special case of common boolean satisfiability problem, which one is NP-complete. Every clause can comprise just two disjunctions.
The problem is described thoroughly at [Wikipedia, 2-satisfiability](https://en.wikipedia.org/wiki/2-satisfiability).
Generally, we have boolean formula expressed in the form of 2-CNF formula. And we need to provide a solution or prove that it is impossible.
[cp-algorithms.com, 2SAT](https://cp-algorithms.com/graph/2SAT.html) is a good source to familiarize yourself with the common notion of solution.
The essential part of the algorithm is a tricky depth-first graph iteration. There is no rocket science but it is still not easy to take into account all data mutations with some specific steps on back propagations. And as it sometimes happens with complex algorithms straightforward implementation can look very entwined. Moreover, most implementations I found didn't provide me a chance to trace code back to algorithms. So I stuck for some period in attempts to understand all aspects. And once functionality was completed and passed grades I tried to bring a more transparent and easy-to-follow solution.

Let's skip all parsing stuff. The entry point for all fun is the implication graph. It is derived from CNF formula with low: `(a ∨ b) == (¬a -> b) ∧ (¬b -> a)`.
Therefore for one clause, we have 2 edges. And every variable is mapped to two nodes (positive and negative). That's how we will have 0 and 1 nodes instead of `a` and `¬a`. 
Graph representation:
```scala
case class ImplicationGraph(nodeCount: Int, adjacencyList: Map[Int, List[Int]])
```
Simple `(a ∨ b)` will be converted to `ImplicationGraph(4,Map(2 -> List(1), 3 -> List(0)))`. 
4 nodes are required to describe all possible values for two boolean variables. 
`(¬a -> b)` means that `¬a`(2) node has one edge to `b`(1) node.
`(¬b -> a)` means that `¬b`(3) node has one edge to `a`(0) node.

All preparations are done and we can move to implementation of nucleus thing - finding strongly connected components. I will use SCC abbreviature for the rest of the post. SCC is an essential term for our goal. After finding SCCs we will be able to verify the initial formula for satisfiability. You definitely need to check out some good lecture to find out formal proof but briefly, it sounds like "non-having `x` and `¬x` nodes allow to state that formula is satisfiable. And to assign correct values we need to just traverse over SCCs in topological order and assign boolean values accordingly to gathered nodes.

It sounds really simple. But we need SCCs. In topological order. [Kosaraju](https://en.wikipedia.org/wiki/Kosaraju%27s_algorithm) and [Tarjan](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm) cover our need. I preferred Tarjan. Initially, it looked more massive but after all, I liked Tarjan's core idea and completed switched to it. 
Really great explanation: [https://www.youtube.com/watch?v=wUgWX0nc4NY](https://www.youtube.com/watch?v=wUgWX0nc4NY)
I am not pretending for an explanation of all aspects so I hope that anyone who reads it familiarized with a high-level idea.

And remainder: we want to implement it in functional fashion.
The most challenging part of implementation is the necessity to update a set of data structures at every iteration. Therefore I decided to isolate such entwined updates inside separate methods. Also, the algorithm is inherently recursive. We need some state to pass through iterations.  Operations over state inevitably depend on the implementation of state (particular data structures). But also we still want to provide a kind of API for iterations steps to make implementation as transparent as possible. Once we have a demand to encapsulate the implementation of the state, it becomes pretty reasonable to join the state with API required to implement the algorithm.
That's how we engender `TarjanTraverseAlgebra`.
Let's go step by step through the API and appropriate interpreter lines.

State handling is defined by the following structure:
```scala
case class TarjanTraverseInterpreter(
                                      currentId: Int,
                                      assignedId: Vector[Int],
                                      visited: BitSet,
                                      stack: Stack[Int],
                                      lowLinks: Vector[Int],
                                      stackIndex: BitSet,
                                      sccs: List[List[Int]]
                                    ) 
```
`currentId` handles the latest assigned id (reminder - we need to assign a new incremental id to be agnostic to the order of initial ids).
`assignedId` is a vector to persist mapping between original and assigned ids.
`visited` is BitSet to store nodes which already during our depth-first iterations.
`stack` is an algorithm-specific stack, we need id to build SCC.
`lowLinks` is another algorithm-specific handler, it is dynamic mapping which finally will contain mapping "node-SCC" and during our iterations, it is used to build current SCC itself.
`stackIndex` is the index for the stack, it allows quickly to check if a node is presented in the stack.
`sccs` is an accumulator for result.

It is a represetation of structure for a particular state handler. To implement an algorithm we need mentioned `TarjanTraverseAlgebra` algebra.

Let's go.
I will put implementation to the same snippets to simplify things.

Easiest part. We just need to have a possibility to know the current id.
```scala
  /**
   * Current current id
   */
  def id: Int

  ...

  override def id: Int = currentId
```

And very core part after it. Iteration over a new node. That means the assignment of the new id to the passed node. That something we does every iteration step: increment id for further iteration, assign a current id to the passed node (original id), push it to stack, set low link (low link is just an id, we fold over lowlinks only during the backpropagation), and we need it to add the node to visited and stack index.
```scala
  /**
   * Increment id, push it to stack and update low link
   *
   * @param at source id
   * @return updated state
   */
  def assign(at: Int): TarjanTraverseAlgebra

  override def assign(at: Int): Self = copy(
    currentId = currentId + 1,
    assignedId = assignedId.updated(at, id),
    stack = stack.push(at),
    lowLinks = lowLinks.updated(at, id),
    visited = visited + at,
    stackIndex = stackIndex + at
  )
```


Nothing to comment.
```scala
/**
   * Is node visited.
   *
   * @param id ud to check
   */
  def isVisited(id: Int): Boolean

  override def isVisited(id: Int): Boolean = visited(id)
```

And that how we will union nodes to one SCC. During the back propagation we need to check if the latest node is in the stack. If so - then we can update the source of edge with the minimum of 2 lowlinks.
```scala
  /**
   * Check if current id equal to it's lowlink value.
   *
   * @param id id to check
   */
  def isCurrentIdEqualsToLowLink(id: Int): Boolean

  def updatedLowValueIfInStack(from: Int, to: Int): Self =
    if (stackIndex(to)) copy(lowLinks = lowLinks.updated(from, math.min(lowLinks(from), lowLinks(to))))
    else this
```

And finally, we need a way to collect SCC. It looks massive a little bit. But in fact, it is just a process of popping out the stack until the fact that we already met the begging of SCC (id of SCC is equal to the smallest node in it).
```scala
  /**
   * Collect strongly connected component. Iteration will stop when first node of scc will be found.
   *
   * @param sccId id of strongly connected component
   * @return updated state with new component
   */
  def collectScc(sccId: Int): TarjanTraverseAlgebra

  def collectScc(sccId: Int): TarjanTraverseAlgebra = {
    case class SccCollectIterationContext(sccAcc: List[Int], stackIndex: BitSet, lowLinks: Vector[Int], stack: Stack[Int])

    @tailrec
    def iterate(ctx: SccCollectIterationContext): SccCollectIterationContext = if (stack.nonEmpty) {
      ctx.stack.pop2 match {
        case (node, stackUpd) =>
          val ctxUpd = SccCollectIterationContext(
            sccAcc = node :: ctx.sccAcc,
            stackIndex = ctx.stackIndex - node,
            lowLinks = ctx.lowLinks.updated(node, assignedId(sccId)),
            stack = stackUpd
          )
          if (node != sccId) iterate(ctxUpd)
          else ctxUpd
      }
    } else {
      ctx
    }


    val processed = iterate(SccCollectIterationContext(List(), stackIndex, lowLinks, stack))
    val sccsUpdated = if (processed.sccAcc.isEmpty) sccs else processed.sccAcc :: sccs
    copy(sccs = sccsUpdated, lowLinks = processed.lowLinks, stackIndex = processed.stackIndex, stack = processed.stack)
  }
```

And finally, we have getters for SCC and lowlinks, but let's omit the code for them.

We have everything we need to finally implement the whole algorithm.

So we have `class TarjanStronglyConnectedComponentsFinder(traverseStateInterpeterInitialState: TarjanTraverseAlgebra)`  with one method:
```scala
def findSccs(adjList: Map[Int, List[Int]]): TarjanTraverseAlgebra = {
    def dfs(sourceNode: Int, state: TarjanTraverseAlgebra): TarjanTraverseAlgebra = {
      val sourceNodeAssignment = state.assign(sourceNode)
      val subgraphAssignment = adjList.getOrElse(sourceNode, List()).foldLeft(sourceNodeAssignment)(
        (state, targetNode) =>
          (if (!state.isVisited(targetNode)) dfs(targetNode, state) else state).updateLowValueIfInStack(sourceNode, targetNode)
      )

      if (subgraphAssignment.isCurrentIdEqualsToLowLink(sourceNode)) {
        subgraphAssignment.collectScc(sourceNode)
      } else {
        subgraphAssignment
      }
    }

    traverseStateInterpeterInitialState.getLowLinks.indices
      .foldLeft(traverseStateInterpeterInitialState)((s, i) => if (s.isVisited(i)) s else dfs(i, s))
  }
```
Step by step:
`val sourceNodeAssignment = state.assign(sourceNode)` - here we updated our state with new node, now it is in stack with incremental id and lowvalue.
`adjList.getOrElse(sourceNode, List()).foldLeft(sourceNodeAssignment)` - here we get list of neighbourghs for our current node and starting to iterate over them.
```scala
(state, targetNode) => (if (!state.isVisited(targetNode)) dfs(targetNode, state) else state).updateLowValueIfInStack(sourceNode, targetNode)
```
Nothing special here, just typical depth-first-search. Let's visit our neghbour in case wi didn'd do it before. After it we need to update lowlinks. Please pay attention it it something that works during the back propagation. That means that we ve visited everything we can until the bottom and after it edge by edge we returning back. And every edge we update source node with the minimum lowlink. That's how we mark all nodes from the same SCC with sole identifier.

That's how we marked all our subgraph (relatively to the current node) and received `subgraphAssignment` state. If during the backpropagation we see that we already at the point where SCC had started we need to just gather nodes and put it to our state. After it, nodes will be visited and out of the stack. That's how we can be sure that any single node belongs to strictly once SCC.

`(s, i) => if (s.isVisited(i)) s else dfs(i, s)` and then we just traverse over all nodes.

Tarjan's part is done at this point.

At this point we can check if source CNF is satisfiable:
```scala
val tarjanSolution = TarjanStronglyConnectedComponentsFinder(implicationGraph.nodeCount).findSccs(implicationGraph.adjacencyList)
val (sccs, index) = (tarjanSolution.getSccs.reverse, tarjanSolution.getLowLinks)

def negation(node: Int) = if (node < cnf.variablesCount) node + cnf.variablesCount else node - cnf.variablesCount

val isSatisfiable = sccs.forall(_.forall(node => index(node) != index(negation(node))))
```
We just returned back to the terms of source boolean variables. `negation` function allows to interpret any node as some value of variable and find opposite node.

To assign values we need to do final step:
```scala
val emptyAssignments = Vector.fill(cnf.variablesCount)(0)

def toPos(postion: Int) = if (postion < cnf.variablesCount) postion else postion - cnf.variablesCount

val boolValAssigned = sccs.foldLeft(emptyAssignments)((assignments, scc) => scc.foldLeft(assignments)((assignments2, node) => {
  val pos = toPos(node)
  if (assignments2(pos) == 0) assignments2.updated(pos, if (node < cnf.variablesCount) 1 else -1)
  else assignments2
}))
```

All code is available at: [https://github.com/antonkw/2CNF-solver](https://github.com/antonkw/2CNF-solver)
