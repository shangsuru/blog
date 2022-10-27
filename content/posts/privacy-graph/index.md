---
title: "Privacy-Preserving Graph Analytics"
description: "Secure Multi-Party Computation combined with large-scale Graph Analysis Frameworks."
tags: [cryptography]
date: 2022-03-22
math: true
---

A huge challenge in today's society is utilizing private data without revealing it, enabling collaboration of distrusting parties on their combined data that would have otherwise been impossible due to privacy concerns or regulations. Secure Multi-Party Computation can solve this problem, keeping sensitive personal data entirely private. A lot of data is represented as graphs, like social network data, and leads to interesting use cases like fraud detection or contract tracing, where privacy plays a significant role. In this post, I focus on privacy-preserving graph analysis by presenting and discussing two recent works by Nayak et al. (S&P'15) [1] and Araki et al. (CCS'21) [2]. For their practicality in real world applications, it is important that the presented protocols are efficient and scale to large amounts of data.

# 1. Introduction

Graphs can be used to efficiently model large varieties of data, from social networks, disease spread or transactions in payment networks. Also, many machine learning solutions like recommender systems run on data in the form of graphs. Since these problems tend to entail large amounts of data, systems have to be designed to be scalable and highly parallelizable. As in all fields of data analysis, also graph analysis often relies on private, sensitive user input. To protect privacy, Secure Multi-Party Computation (MPC) can be used. One main challenge hereby is to make the employed protocols highly parallelizable, so that the solution scales to the massive size of the input data sets, while not leaking anything about the data or even the topology of the graph.

*Outline.* In §2, I define the notion of MPC along with its security definition and give an introduction to a prominent example of a MPC technique: Yao's Garbled Circuits. In §3 and §4, I summarize the contributions of Nayak et al. [1] and Araki et al. [2], respectively. 
§5 outlines the limitations of the current privacy-preserving graph analytics frameworks and offers pointers to possible future research directions.


# 2 Background

I want to start by introducing Multi-Party Computation (MPC) including its security properties and the two typically considered threat models, cf. §2.1. In this blog post, I focus on Yao's Garbled Circuits Protocol, which is used as backend in [1].
Then, I outline the programming abstractions of two prominent frameworks for parallel processing on graphs, Pregel [3] and Graphlab [4], in §2.2. The two papers that I want to present in § 3 and § 4 lie in the intersection of MPC and graph analysis by designing a privacy-preserving graph analytics framework using MPC.

## 2.1 Multi-Party Computation

Secure Multi-Party Computation (MPC) enables a group of mutually distrusting parties to compute a joint function that depends on their private inputs without revealing anything but the result. MPC has come a long way from the 1980s, where the underlying theories were developed that showed that it is possible to compute an arbitrary function in a privacy-preserving manner, to today's state-of-the-art techniques with significant improvements in terms of efficiency. Thus, nowadays, MPC is fast enough to be applied in a wide range of practical applications. It enables privacy-preserving computations on highly sensitive data, like location, health, or social data.

### 2.1.1 Ideal Functionality}

The security notion of MPC protocols can be put in very simple terms: the *real-ideal paradigm* [5].
In an ideal world, there exists one incorruptible party that everybody fully trusts. A secure computation can, hence, be realized very easily: Everyone sends his private input to the trusted party via secure channels. The trusted party computes the function of all inputs and sends back the result. Privacy is trivially ensured, because each party only sees its input and the result. 

The notion of a fully trusted third party is imaginary, but a secure MPC protocol simulates such an ideal world. In the real world, there is no trusted party. Instead, all parties communicate with each other according to the MPC protocol. Adversaries, however, can corrupt parties. Their capabilities depend on the threat model:
A semi-honest adversary follows the protocol as specified, but tries to learn as much as possible from the exchanged messages. That is why those types of attackers are also referred to as *honest but curious*. In contrast, a malicious adversary does not stay passive, but can arbitrarily deviate from the protocol to violate security by injecting and manipulating  messages. The real world protocol can be considered secure, if all an adversary can achieve is already possible in the ideal world.

### 2.1.2 Yao's Garbled Circuits

Yao's Garbled Circuits is an example for a generic MPC protocol. That means, unlike specialized MPC protocols that only realize a distinct functionality, like Private Set Intersection, it can compute any discrete function that can be represented as a fixed-size boolean circuit. It runs in a constant number of rounds and communication and computation costs are linear in the size of the circuit. It works like the following:

A boolean gate has two input wires and one output wire. Each of the two input wires is associated with two input keys for the possible input bits 0 and 1, and analogously the output wire is associated with two output keys, where one represents the output of 1 and the other the output of 0. One of the parties, called the *garbler*, selects those keys and constructs an encrypted look-up table for each of the function's boolean gates. Within the look-up tables, the output bits are replaced by its output key and each output key gets encrypted with the combination of the two input keys corresponding to the two inputs that result in the given output.

As an example, consider an AND gate, with two input wires $i$ and $j$ and an output wire $t$. The associated keys would then be $k_i^0$, $k_i^1$ for the left input wire, $k_j^0$ and $k_j^1$ for the right input wire and $k_t^0$ and $k_t^1$ for the output. The resulting table T is depicted in Equation 1.

![Equation 1](posts/privacy-graph/images/3.png)

The garbler can send the table to the second party, called *evaluator*, together with his chosen input key and the input key that corresponds to the choice of the evaluator, which is determined via Oblivious Transfer. In this way, both parties do not learn the chosen input bit of the other party. The evaluator can now use both input keys to decrypt exactly one row in the table correctly to obtain the associated output key. The output keys are appended with a certain number of trailing zeros, so that they can be differentiated from random strings and the evaluator knows that the right row was decrypted. Note that when the garbler permutates the order of the rows in the look-up table beforehand, the evaluator cannot deduce the corresponding output bit. Following in that manner, a whole boolean circuit can be evaluated recursively. The garbler's operations are easily parallelizable, but evaluating the garbled circuit must be done layer by layer and therefore, the degree of parallelization during the evaluation depends on the depth of the circuit.

### 2.1.3 Challenges

The major challenge for further adoption of secure computation is that nearly all use cases regarding data analysis involve very large datasets and complex operations while Yao's Garbled Circuits has communication overhead for every simple AND gate. Secure computation solutions need to embrace parallelism to be able to scale efficiently over a huge number of compute nodes and with it provide easy to use abstractions that make those techniques accessible even to programmers with little to no cryptographic expertise.

## 2.2 Programming Paradigms for Graph Analysis

Many practical computing problems are concerned with large graphs. Computations on social networks, disease spread analysis and also many data mining and machine learning problems, e.g., for recommender systems, can be expressed using graphs. Concrete algorithms run on graphs include shortest path computations, clustering, connected component identification, or variations of Page Rank.

Many problems require large-scale graphs for which we need efficient processing. The computation needs to be parallelized and distributed over multiple machines. For this purpose, several distributed programming frameworks have been developed. They provide an easy-to-use API that is expressive enough for implementing arbitrary graph algorithms, but hides the complexities of the parallel and distributed design. 

In the following, I represent the simple programming abstraction for parallel computations on graphs, found in works like Pregel [3] and Graphlab [4], that also underpins the two papers I want to present. Operations are carried out on a data-augmented graph, that is a directed graph with user-defined data on each vertex and edge. The three operations used are:

1. Apply: Vertices apply a user-defined function $f_A$ on their data and update it with the result.
2. Gather: Vertices aggregate the data of their incoming edges, via a user-provided aggregation operator $\oplus$, and update their own data with it.  The aggregation operator has to be commutative and associative, so that the result of the aggregation is not dependent on the ordering of the edges.
3. Scatter: Vertices propagate their data to all outgoing edges, and update the edges data according to some user-specified function $f_S$.

In this way, each vertex can perform computation on its own data and the data of its adjacent edges in parallel with the other vertices, providing a suitable interface for parallel graph computations. What is missing is that those operations should also be carried out in a privacy-preserving manner, hiding both inputs and the topology of the underlying graph. Solutions to this are described in the next section.

# 3 GraphSC: Parallel Secure Computation Made Easy

In this section, I outline the works of [1]. The focus is on how the discussed Scatter, Gather and Apply operations are realized in an oblivious manner and how they are parallelized. The graph algorithms are considered oblivious, if their accesses to memory locations are the same for any two input graphs. That means, the algorithms have to be completely incognizant of the underlying graph structure.


GraphSC is a parallel secure computation framework that combines efficient computation on graphs with a secure computation backend based on garbled circuits. It offers similar programming abstractions to popular graph processing frameworks like Pregel [3] and GraphLab [4], introduced in §2.2, namely Scatter, Gather, and Apply operations, with which it can support a broad range of data mining algorithms. GraphSC implements these operations in a data oblivious way to prevent any information leakage. This is challenging, because even the graph structure itself should remain private. However, note that the number of times a vertex is accessed during a Scatter operation, for example, already reveals how many neighbors that vertex has. Its second contribution is to also develop parallelized versions of those oblivious algorithms, so that the computation can scale for large graphs. Hereby, it focuses on Two-Party Computation, i.e., secure computation among exactly two parties (§ 2.1) in the semi-honest model (§ 2.1.1). The secure versions of Apply, Scatter, and Gather have a small logarithmic overhead, together with minimal communication overhead and an insignificant impact on accuracy. 

## 3.1 Single-Processor Oblivious Algorithms

I start by discussing the oblivious variations of algorithms for Scatter, Gather, and Apply in the single-processor setting. To hide the graph structure, the graph needs to be represented in a way that does not disambiguate between edges and vertices. I, henceforth, treat a graph as a list of data tuples of the same size containing a bit that tells if the tuple represents an edge or a vertex.


1. Apply is straightforward: Make a linear scan over the list and apply the function $f_A$ to each vertex tuple and a dummy operation on each edge tuple.
2. For Gather, we first perform an oblivious sort, so that each vertex appears immediately after the list of all its incoming edges (*destination sort*). Then, it follows an *aggregate* operation: In a linear scan over the list, we update each value of a vertex with the sum of the longest preceding sequence of edge tuples, i.e., all incoming edges to that vertex, using the aggregation operator $\oplus$.
3. Scatter is very similar to the Gather operation. First, we perform an oblivious sort, so that each vertex is immediately followed by all its outgoing edges (*source sort*). Then, we do a *propagate* operation, i.e., do a single linear scan and update each edge's data with the nearest preceding vertex, i.e., the vertex it originates from, by applying the $f_S$ function.

Let $M := |V| + |E|$. When we assume that each $f_A$, $f_S$, and $\oplus$ are of $O(1)$ cost, then, Apply can be performed in $O(M)$ time. Oblivious sort has $O(M\log\ M)$ time complexity. Both aggregate and propagate operations take $O(M)$ time. Therefore, Scatter and Gather each run in $O(M\ log\ M)$ time.

## 3.2 Parallel Oblivious Algorithms

Now we consider N processors that make oblivious accesses to shared memory and describe how we can parallelize the oblivious algorithms for Scatter, Gather, and Apply. We set the number of processors to be the optimal N := |V| + |E|. Descriptions for the algorithms with a smaller number of processors can be found in the original paper [1], in §3E. Recall again that the difficulty in the design of the parallelized algorithms is that each processor should access the tuples in the list in the same way, independent on the structure of the graph, i.e., if it is a vertex or edge or how many edges a vertex has.


1. Apply can be trivially parallelized, each processor is assigned a tuple and applies $f_A$ or a dummy operation, depending on if it is a vertex or an edge.
2. Gather consists of an oblivious sort and an aggregate operation. Oblivious Sort is a $log(|V| + |E|)$-deep circuit [6] and can, therefore, be trivially parallelized at the circuit level. Hence, we focus on how to parallelize the aggregate operation. Instead of doing a linear scan like in the sequential version, each processor is assigned one tuple in the list and needs to compute the sum of the longest prefix of edges preceding that tuple using the aggregation operator $\oplus$. Again, the longest preceding prefix of edges would be the list of all incoming edges, after having performed the oblivious *destination sort*. If the tuple that the processor got assigned to is a vertex, it can update its data with the computed sum. The way the processors compute the sum over the longest prefix of edges is a bottom-up approach: At time step $\tau$, each processor only computes the sum over the immediately preceding segment of tuples of size $2^\tau$. That is, at $\tau = 1$, it only computes the sum over the two preceding tuples, at $\tau = 2$ over the 4 preceding tuples, a.s.o, until it covered the total length of the list, in $log(|V| + |E|)$ steps. Note that the sum over each segment, e.g., of size 8, can be determined by combining the already computed values of the two segments of size 4 that form the left and right half of the larger segment: Either aggregate both their sums if no vertex appeared yet in the right half, or take the sum of the right half as the result. 
3. For Scatter, similarly, we only need to focus on how to parallelize the propagate operation. To repeat, a propagate operation updates each edge with the data of the nearest preceding vertex, which is the vertex that edge is originating from after having performed the oblivious *source sort*. Conveniently, a propagate operation can be expressed as an aggregate operation, when every edge stores the value of the preceding vertex if a vertex immediately precedes, and $-\infty$ otherwise. The aggregation operator is then defined to be the *max* operator. So, after $log(|V| + |E|)$ steps, each processor has computed the value of the nearest vertex and can update the value of its assigned tuple, if it is an edge.

I now summarize the complexities for the parallel oblivious algorithms for each processor. For Apply, it takes $O(1)$ since every processor only evaluates a single $f_A$ or a dummy operation. Both Scatter and Gather are $O(log |V| + |E|)$ time. This represents a considerable blowup compared to the parallel time of insecure Scatter and Gather, which is $O(1)$ and $O(log d_m)$, respectively, where $d_m$ denotes the maximum degree of a vertex in the graph.

## 3.3 Parallel Secure Algorithms

The reduction from parallel oblivious algorithms to parallel secure algorithms can now be simply achieved by making use of a garbled circuit backend. The oblivious algorithms are represented as a circuit. One party acts as the garbler and the other as the evaluator, while the sensitive data is secret shared between the two parties. Each party parallelizes the computational task, garbling and evaluating the circuit, across its processors.

# 4 Secure Graph Analysis at Scale

The second paper that I want to present is by Araki et al. [2] and also deals with highly-scalable secure computation of graph algorithms. It directly builds upon ideas from the previous paper by Nayak et al. [1] that I discussed in §3. Their major contribution is to derive an optimization of the Gather and Scatter operations, which originally made use of secure sorting (destination and source sort, respectively) for obliviousness (§3.1). They managed to replace those with a more efficient secure shuffling protocol. I will discuss the general idea in §4.1. Moreover, they construct and efficiently implement protocols for secure shuffling and sorting, cf. §4.2. In contrast to Nayak et al., their protocols are based on the 3-server setting with an honest majority, with either semi-honest or full security. I show ideas for extensions to full security in §4.3. A more detailed comparison with respect to security guarantees, computation, and communication efficiency between the two papers follows in §4.4.

## 4.1 Optimization for Oblivious Graph Algorithms

To repeat, for obliviousness, the graph is represented as a list of tuples $T$, where each tuple represents either an edge or vertex. Scatter, the operation that propagates data from a vertex to all its outgoing edges, requires the tuples $T$ to be sorted so that each vertex is immediately followed by all its outgoing edges. This sorting order is called *source sort*. Gather, the operation that takes the values of all incoming edges and updates the data of the vertex, requires a *destination sort*, where all incoming edges appear before their respective vertex, cf. §3.1. Source and destination sort are the only two sort orders required. In order to alternate between a series of Gather and Scatter operations, they have to be recomputed. The key insight is that it is sufficient to compute these two orders only once during an initialization phase and switch between them using a secure shuffle.

![Initialization Phase](posts/privacy-graph/images/4.png)
<p align = "center">
Fig.1 - Initialization Phase
</p>

During an initialization phase, the servers compute a secure shuffle A of their inputs and another secure shuffle from shuffle A to shuffle B. The servers maintain the necessary information to be able to jointly recompute the secure shuffle from A to B and vice versa. Next, A is sorted with the source sort order for Scatter operations and B is sorted with the destination sort order for Gather operations. The two corresponding permutations can be made public, because they reveal no information about the relation between the original inputs and the sorted list. Computing one secure shuffle between A and B and the respective permutation to the required sorting order is much more efficient than repeatedly sorting a list of tuples securely, cf. §4.4. 

## 4.2 Secure Sorting And Shuffling

Now, I want to provide more information on the the protocols used for secure sorting and shuffling. The sorting protocol was proposed by Hamada et al. [7]. It consists of two phases, where the first phase is to compute a secure shuffle of the list of tuples to sort. In the second phase, a comparison-based sorting protocol is applied on the result. The output is a permutation that maps the input to the sorted order, which can be made public, because its input was the random shuffle of the data, which does not leak any information about the original list of tuples. The sorting protocol has to be run only once. Subsequently, the resulting permutation function can be used to bring the data into the desired sorted order. The method for secure shuffling is based on Laur et al. [8]. It is a 3-round shuffle protocol (cf. §4.2.1. Araki et al. also propose a transformation into a new 2-round shuffle protocol (cf. §4.2.2). 

### 4.2.1 3-round shuffle protocol

The input to the shuffle protocol is a sharing of a list of tuples $T = A \oplus B \oplus C$. The goal is to produce a new sharing $T_O = A_O \oplus B_O \oplus C_O = \pi(T)$, which is a random permutation of $T$. To ensure that none of the participating servers S1, S2, or S3 can learn the permutation $\pi$, the basic shuffle protocol is constructed such that the first two servers generate a random permutation, which the third server does not learn. This basic protocol has to be repeated three times. In each round another server does not learn the permutation, such that in the end, none of the servers can single-handedly reconstruct the permutation $\pi$:

![Equation 2](posts/privacy-graph/images/5.png)

In the first round, S3 does not learn the random permutation chosen by S1 and S2, in the second round S1, and in the third round S2. We will now go through the basic protocol (cf. Fig. 2) in the first round:

![Basic Shuffle Protocol](posts/privacy-graph/images/1.png)
<p align = "center">
Fig.2 - Basic Shuffle Protocol
</p>

1. The list of tuples $T$ is shared in a 2-out-3 sharing. Each server knows two out of the three shares $A$, $B$, and $C$. At the end of the protocol, each server should learn two out of the three output shares $A_O$, $B_O$, and $C_O$. The output shares combined will be chosen to yield a permutation of the original list of tuples:  $A_O \oplus B_O \oplus C_O = \pi(A \oplus B \oplus C) = \pi(T)$.
2. Each pair of servers $S_i$, $S_j$ holds a common secret $s-ij$.
3. S1 and S2 use their shared secret $s-12$ to derive a permutation function $\pi$ and use it to compute a shuffle of their respective shares. S3 does not know $s-12$ and, therefore, does not know the shuffle of the individual shares.
4. S3 uses its two secrets, $s-23$ and $s-31$, to derive two random values that also serve as two of the output shares, $C_O$ and $A_O$. $A_O$ can be computed by S1, and $C_O$ can be computed by S2, because they share the respective secret with S3.
5. $A_O$ is used by S1 as a masking value to send the shuffle of $A$ to S2. S2 does the same with the shuffle of C and $C_O$.
6. Now both can compute $B_O$. Highlighted in yellow, one can see that $B_O$ contains the permutation of all three shares $A$, $B$, and $C$, which is revealed when XORing $B_O$ with the two other shares $A_O$ and $C_O$.
7. The result is that each server knows two out of the three shuffled output shares, which combined yield a permutation of the initial list of tuples $T$. Moreover, only S1 and S2 know the permutation function $\pi$ that created the shuffled shares.

With the need to execute this basic shuffle protocol three times and 2 message exchanges per invocation, this yields a 3-round, 6-message, shuffle protocol in the semi-honest setting. An extension for full security will be discussed in §4.3. 

![2-Round Shuffle Protocol](posts/privacy-graph/images/2.png)
<p align = "center">
Fig.3 - 2-Round Shuffle Protocol
</p>

### 4.2.2 2-round shuffle protocol

Araki et al. also present a more efficient 2-round, 4-message, shuffle protocol for which no extension to full security is given (cf. Fig. 3):

1. Again, the list of tuples is shared in a 2-out-of-3 sharing between the three servers. Each pair of servers $S_i$, $S_j$ shares a common secret $s-ij$.
2. The servers use their secrets to generate permutations $\pi-ij$, random values, $\tilde{A}$ and $\tilde{B}$, that serve as output shares later, and values $Z-ij$ used for masking permutations of the shares A, B, and C, that need to be exchanged between the servers.
3. The goal is to generate shares $\tilde{A} \oplus\tilde{B} \oplus \tilde{C} = \pi_{23} \circ \pi-31 \circ \pi-12(A \oplus B \oplus C)$. Note that none of the servers knows all three permutations $\pi-ij$, so the result is a secure shuffle. In the subsequent steps, the servers apply the permutations $\pi-ij$ to their shares (highlighted blue for $\pi-12$, red for $\pi-31$, and yellow for $\pi-23$ in Fig. 3) and apply the respective masking value $Z-ij$ before sending them to other servers. Each masking value $Z-12$, $Z-31$, and $Z-23$ is used by two servers, ensuring that they cancel out when the individual shares are combined at the end.
4. After all these steps, S2 has received $X_3$, which contains $\pi(A \oplus B)$ with all three masking values Z. S3 received $Y_3$, which contains $\pi(C)$ with all three masking values Z.
5. Both have to apply one of the random output shares $\tilde{B}$ and $\tilde{A}$ respectively and when the results are put together, this yields the last output share $\tilde{C}$, where all masking values $Z$ cancel out and $\tilde{A} \oplus\tilde{B} \oplus \tilde{C} = \pi(A \oplus B \oplus C)$ as intended.

## 4.3 Extensions For Full Security

In the following, I briefly describe the capabilities of a malicious adversary for the 3-round 6-message shuffle protocol in Fig. 2 and outline how the previously discussed protocol can be adapted to be secure in this security model, according to Araki et al. A malicious adversary is only able to alter one of the exchanged messages, e.g., $\pi(A) \oplus A_O$. We denote this change as $\pi(A) \oplus A_O \oplus \Delta$. This $\Delta$ propagates into the output, hence, produces $T_O \oplus \Delta$. No information is leaked, but the output $T_O$ is not a valid shuffle of the input $T$. To defend such a manipulation, it is necessary to run a set equality verification that detects if $T$ and $T_O$ contain the same set of tuples with overwhelming probability. The equality check has to be run after each invocation of the basic shuffle protocol, otherwise an attacker can gain information through selective failure attacks, for details refer to the full paper of Araki et al [2].

The protocol runs $\kappa$ tests, where in each test, a different set of tuples (rows) and fields in the tuples (columns) are selected to compare $T$ and $T_O$. For this purpose, each row is extended by $\kappa$ random bits that serve as identifier of the rows and, in each round, determine if the corresponding row is selected. The servers also choose a random subset of columns to compare and, then, compute the XOR over those rows and columns to verify that they are equal. This procedure yields a probability of detecting a difference between $T$ and $T_O$ of $1 - (3 / 4)^{\kappa}$. The full security proof can be found in the paper by Araki et al [2].  

## 4.4 Improvements over GraphSC

The discussed protocols operate in the 3-party honest-majority setting, which is an improvement performance wise over the 2PC protocols of GraphSC, but requires a stronger trust assumption. Moreover, in contrast to GraphSC, it is also discussed in more detail how the protocols can be extended to counter malicious adversaries, cf. §4.3.

The biggest advancement is the idea of substituting sorting with shuffling. In GraphSC, to execute every Scatter and Gather operation, the tuples have to be repeatedly sorted to source sort order and destination sort order, respectively, cf. §3.1. Araki et al. manage to replace sorting with shuffling by only computing the two sort orders in a first round and use shuffling to switch between the two orders in subsequent rounds, cf. §4.1. Alternating between the two sort orders has the largest influence on the overall runtime, because it is used repeatedly for every Gather and Scatter operation. Therefore, replacing sorting with the much faster shuffling results in a huge performance boost. Secure sorting is an $O(n\ log\  n)$ algorithm, while the runtime of shuffling is close to being linear. Their analysis shows that sorting is by far the slowest operation. It is about ten times slower than secure shuffle in their evaluation. 

# 5 Limitations and Future research

I want to conclude this post by identifying the limitations of the discussed works about privacy-preserving graph analytics that I have noticed so far, cf. §5.1. Lastly, I would like to point out potential directions for future research as well, cf. §5.2.

## 5.1 Limitations and Drawbacks

The following are limitations and drawbacks of the examined privacy-preserving graph analytics frameworks compared to frameworks that operate on plaintext.

- *Cost of obliviousness.* The graph operations discussed in §2.2 have to be implemented obliviously. This leads to a blowup of the required total work, as disussed in §3.2. Moreover, the programming API has to be restricted compared to similar plaintext frameworks like Pregel [3]: Arbitrary message passing between vertices is no longer possible as this would break obliviousness.
- *No optimizations based on the graph topology.* The main security requirement of frameworks for privacy-preserving graph analytics is that they have to be completely incognizant about the structure of the graph. However, there is a large variety of plaintext graph algorithms that are optimized for specific types of graph structures. Those graph algorithms are particularly efficient if the graph has certain properties. For example, for finding a minimum spanning tree, Prim's algorithm is faster for dense graphs, whereas Kruskal's algorithm works better with sparse graphs. GraphSC and related frameworks cannot employ such optimizations that depend on the knowledge of the graph structure. 
- *Separate Scatter and Gather phases.* Although Scatter and Gather operations can itself be run in parallel, the computations always have to switch between subsequent Scatter and Gather phases, i.e., Scatter and Gather operations cannot run concurrently. This is because each require the list of tuples of the graph to be sorted in a different order, source sort and destination sort, respectively, to be carried out obliviously. To switch between the sorting orders, the servers have to jointly compute a secure shuffle of the tuples, cf. §4.1. This is a limitation of the general design of the oblivious algorithms used. In plaintext systems, an interleaving of Scatter and Gather operations would be possible and could lead to better performance due to higher degrees of concurrency.
- *High communication overhead.* Each Gather and Scatter operation requires communication when executed as a secure circuit, which is a huge bottleneck of the system and a major reason why privacy-preserving graph processing frameworks cannot keep up with the performance of their plaintext counterparts. 
- *Handling of large graphs.* GraphSC makes use of an XOR-based secret sharing scheme of the entire graph. That means each server's share still corresponds to the size of the original graph. This neglects the reality of common graph analytics applications, where the size of the graph commonly exceeds the capacities of a single machine's main memory. For example, the Facebook Graph consists of about one billion nodes and 200 billion edges. There is currently no way to partition a large graph over multiple machines.

## 5.2 Future Research Directions

I identified two possible directions for future research in the area of privacy-preserving graph analytics. One idea is to incorporate homomorphic encryption, which does not exhibit the high communication overhead of MPC protocols. Using homomorphic encryption, the graph could be partitioned across multiple machines along clusters (§5.2.1). A second idea is to allow for dynamic updates of the graph topology during computation, which is a current research area for plaintext graph analytics, known as *graph streaming* (§5.2.2).


### 5.2.1 Utilizing homomorphic encryption
    
Communication between garblers and evaluators during the MPC computation was identified to be a key bottleneck of system performance by Nayak et al. [1]. Communication between the servers is necessary for every single Scatter and Gather operation. In the future, with further hardware improvements, communication will become more and more the bottleneck over computation. Current techniques for homomorphic encryption, computing on encrypted data without the communication overhead of MPC protocols, might also improve. This could make it feasible to utilize homomorphic encryption for private graph analytics as well. One could first find an efficient partitioning of the graph, e.g., via clustering, in a preprocessing phase. Clusters are the very dense areas of the graph, where Scatter and Gather operations would naturally have a high communication overhead. These clusters can then be assigned to different servers, which could perform the designated computation obliviously on encrypted data, using homomorphic encryption. This would only require communication across clusters, which is negligible when a good partitioning of the graph was found. Graphs that are too large for a single machine to handle could also be managed using this method, due to the partitioning of the graph across multiple machines.
   
### 5.2.2 Enabling dynamic graph changes 
    
GraphSC does not support modification of the graph structure during computation, unlike Pregel [3]. This inhibits analyzing the evolution of a dynamically changing graph topology over time, as it is the case in, e.g., social networks, where the graph could get updated by a continuous stream of edge modifications. Hereby, a new edge corresponds to a new connection between users, while the removal of an edge means that there is no longer a connection. For many more applications graphs are not only very large, but also dynamic, which led to the emergence of new graph streaming frameworks like STINGER [9] and Aspen [10] that are executing graph analytics algorithms concurrently with updates on the graph, e.g., the insertion of edges. For a broad overview of the field of graph streaming frameworks for the analysis of dynamic graphs, see Besta et al. [11]. The incorporation of graph streaming into privacy-preserving graph processing frameworks would allow for dynamic updates of the graph topology and data without restarting the computation from the beginning. 


# 6 Conclusion

I showed that the work by Araki et al. [2] offers a significant performance improvement over the original work by Nayak et al. [1] by substituting secure sorting with secure shuffling for all invocations except from a one-time initialization phase.
Due to their ability to highly parallelize the graph computation, it becomes feasible for a wide range of graph problems to run the analysis securely on private data. However, many interesting applications of graph analysis in practice come from the Big Data world, where graphs can consist of billions of edges and become too big for a single machine to store. The privacy-preserving techniques presented clearly still have their limitations compared to the current state-of-the-art technologies designed for massive amounts of plaintext data. Thus, further performance improvements of the underlying protocols are necessary. One direction might be to dynamically incorporate changes of the graph structure during analysis, following the approach of graph streaming frameworks [11]. Current privacy preserving frameworks for graph analysis cannot handle dynamic updates of the graph topology, but, e.g., for social networks or graphs that model disease spread or transactions in a payment network, the graph structure is always under constant change.

# References

[1] Kartik Nayak, Xiao Shaun Wang, Stratis Ioannidis, Udi Weinsberg, Nina Taft, and Elaine Shi. Graphsc: Parallel secure computation made easy. Symposium on Security and Privacy, 2015.

[2] T. Araki, J. Furukawa, B. Pinkas, K. Ohara, H. Rosemarin, and H. Tsuchida. Secure graph analysis at scale. CCS, 2021.

[3] Grzegorz Malewicz, Matthew H Austern, Aart JC Bik, James C Dehnert, Ilan Horn, Naty Leiser, and Grzegorz Czajkowski. Pregel: a system for large-scale graph processing. ACM SIGMOD International Conference on Management of Data, 2010.

[4] Yucheng Low, Joseph Gonzalez, Aapo Kyrola, Danny Bickson, Carlos Guestrin, and Joseph M Hellerstein. Distributed graphlab: A framework for machine learning in the cloud. PVLDB, 2012.

[5] Yehuda Lindell. How to simulate it–a tutorial on the simulation proof technique. Tutorials on the Foundations of Cryptography, 2017.

[6] Miklós Ajtai, János Komlós, and Endre Szemerédi. An 0 (n log n) sorting network. ACM Symposium on Theory of computing, 1983.

[7] Koki Hamada, Ryo Kikuchi, Dai Ikarashi, Koji Chida, and Katsumi Takahashi. Practically efficient multi-party sorting protocols from comparison sort algorithms. International Conference on Information Security and Cryptology, 2012

[8] Sven Laur, Jan Willemson, and Bingsheng Zhang. Round-efficient oblivious database manipulation. International Conference on Information Security, 2011.

[9] Guoyao Feng, Xiao Meng, and Khaled Ammar. Distinger: A distributed graph data structure for massive dynamic graph processing. IEEE International Conference on Big Data, 2015.

[10] Laxman Dhulipala, Guy E Blelloch, and Julian Shun. Low-latency graph streaming using compressed purely-functional trees. ACM Conference on Programming Language Design and Implementation, 2019.

[11] Maciej Besta, Marc Fischer, Vasiliki Kalavri, Michael Kapralov, and Torsten Hoefler. Practice of streaming processing of dynamic graphs: Concepts, models, and systems. IEEE Transactions on Parallel and Distributed Systems, 2021.