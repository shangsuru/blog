---
title: "Verification of Neural Networks"
description: "Applying formal methods to make neural networks safer and more reliable"
tags: [formal-methods, machine-learning]
date: 2023-05-10
math: true
---

Formal methods have proven to be successful for the verification of traditional systems. It is an ongoing research effort to apply them to deep learning based systems, which are being used more and more in safety-critical environments like autonomous driving. In this paper we present the theoretical foundations and state-of-the art tools for two common approaches to neural network verification. 


# Introduction

As machine learning techniques are increasingly being used to solve a variety of problems such as recommender systems, machine vision, and autonomous driving, there are concerns about the lack of methods and tools to provide formal guarantees about the behavior of resulting systems. This is particularly critical for safety-critical applications such as robotics, cybersecurity, and cyber-physical systems. Specifically, DNNs emerge as a cornerstone of automotive software engineering, as it is the key technology that enables vehicular perception and autonomous driving.

However, developing systems with DNNs introduces novel challenges for safety assessments. Their behaviour cannot be fully guaranteed to be robust against adversarial attacks or perturbations in the inputs. Therefore, there is a growing interest in applying formal methods for the verification and validation of safety-critical systems that rely on machine learning, with a focus on DNNs. 

One concrete example of the challenges that arise when using DNNs for safety-critical applications is the classification of stop signs. Researchers have shown that slight modifications of stop signs, such as adding stickers or graffiti, can cause DNNs to misclassify them as other objects, such as speed limit signs. This shows that DNNs can be surprisingly sensitive with respect to unexpected perturbations to their input and highlights the need for formal verification methods to ensure that DNNs are reliable in safety-critical scenarios. As machine learning techniques continue to be used in increasingly important applications, such verification methods will become even more crucial for ensuring the safety and security of our systems and human beings.

Two approaches that have been proposed are constraint-based and Abstract Interpretation-based solutions. While both have shown promise, Abstract Interpretation-based solutions have been demonstrated to scale better for larger neural network architectures, despite being incomplete. This paper aims to provide an overview of these approaches and their potential for neural network verification. 

# Deep Neural Networks

Deep Neural Networks (DNNs) are a class of artificial neural networks that have become increasingly popular in recent years due to their ability to learn complex features from data and achieve state-of-the-art performance on a variety of tasks. DNNs are composed of multiple layers of interconnected nodes, with each node performing a nonlinear transformation on its inputs.

![Architecture of a Neural Network](posts/formal-dnn/images/dnn.png)
<p align = "center">
Fig.1 - Architecture of a Neural Network
</p>

The input layer of a DNN receives the raw input data, such as an image or a sequence of words. The input is then passed through a series of hidden layers, where each layer consists of multiple nodes that compute a weighted sum of their inputs and apply a nonlinear activation function, such as the Rectified Linear Unit (ReLU). The output of one layer serves as the input to the next layer, allowing the network to learn increasingly complex and abstract features as it progresses through the layers (see Figure 1).

Training a DNN involves updating the weights of the connections between the nodes to minimize a loss function that measures the discrepancy between the network's output and the desired output. This is typically done using an optimization algorithm called backpropagation, which involves computing the gradient of the loss function with respect to the weights of the network and updating the weights in the direction of the negative gradient. Backpropagation is an efficient way to compute the gradients of the loss function with respect to all of the weights in the network using the chain rule of calculus.

In recent years, DNNs have achieved state-of-the-art performance on a wide range of tasks, including image and speech recognition, natural language processing, and game playing. However, the use of DNNs also comes with some challenges, such as the need for large amounts of data and computational resources for training, as well as the potential for overfitting and adversarial attacks. Despite these challenges, the potential applications of DNNs are vast, and they continue to be an active area of research and development in the field of machine learning.

Convolutional Neural Networks (CNNs) are a specialized class of deep neural networks that have been highly successful in the field of computer vision, particularly in image recognition tasks. CNNs are designed to automatically learn spatial hierarchies of features from images, allowing them to recognize objects in images with high accuracy.

The architecture of a CNN consists of multiple layers, including convolutional layers, pooling layers, and fully connected layers. The convolutional layers perform a convolution operation on the input image with a set of learnable filters or kernels, which slide over the image and produce a feature map that captures local patterns or features. The pooling layers downsample the feature map by summarizing nearby activations, reducing the spatial dimensionality of the data while preserving important information. Finally, the fully connected layers combine the high-level features extracted by the convolutional and pooling layers into a classification output.

The key advantage of CNNs is their ability to automatically learn hierarchical features from images, without the need for hand-engineered features. This is made possible by the convolutional layers, which learn filters that are sensitive to specific patterns or features in the input data, such as edges or corners. These filters are then combined in subsequent layers to learn more complex features, such as textures and shapes.

CNNs have been highly effective for image recognition tasks, achieving state-of-the-art performance on benchmarks such as ImageNet. In addition to image recognition, CNNs have also been applied to other computer vision tasks, such as object detection, segmentation, and image captioning.

## Modeling DNNs as Graphs

For subsequent sections, it is helpful to model a DNN as an acyclic graph $G = (V, E)$. At each node a function $f_v$ is computed that could be a weighted sum or a activation function and that takes in the values of all the input nodes and computes a single output value. The edges link outputs of nodes to inputs of other nodes. We denote $V^{in} \subset V$ as the non-empty set of input nodes and $V^{o} \subset V$ as the non-empty set of output nodes of the neural network.

## Correctness Properties

When verifying neural networks, there are several possible correctness properties to consider, including robustness and safety. Robustness means that small changes to the inputs should not affect the output. This is important for safety and security, as lack of robustness can lead to errors in classification tasks, with severe consequences. For example, in autonomous vehicles, a vandalized stop sign could cause an autonomous vehicle to miss it. Safety is a class of correctness properties that aims to prevent programs from reaching a bad state. In the context of a neural network collision avoidance system, safety means making the correct decision when facing a potential collision, such as turning right when another vehicle is approaching from the left.

In the following sections we will focus on robustness. What we need is a language for formally specifying the properties we want the neural network to have. The specification will have the following form:

![Correctness Property](posts/formal-dnn/images/pcpc.png)

This way we can also express the desired robustness properties. Robustness is a broad class of properties though. We can specify robustness with respect to various factors like change of brightness, contrast, rotation, etc. For a concrete example, Figure 2 shows a hand-written 7 from the MNIST dataset along with two modified versions: One has its brightness increased and on the other one a spurious dot is added. Despite the minor changes, all images should be classified as 7 to satisfy the robustness property.

![MNIST image with different perturbations](posts/formal-dnn/images/mnist.png)
<p align = "center">
Fig.2 - MNIST image with different perturbations
</p>

We can use our notation in the following way:

![Robustness Property](posts/formal-dnn/images/robustnessproperty.png)

Which norm we use depends on the type of robustness we want to verify. For example, we could use the following two norms: The Euclidean norm and the Maximum norm

![Norms](posts/formal-dnn/images/norms.png)

The Euclidean norm can be understood as the straight line between two images in $R^n$, while the maximum norm is the largest discrepancy between two images. If we want to represent the set of all images that are like c, but where each pixel differs by a certain brightness amount, then we can use the maximum norm in the precondition. This is because the maximum norm captures the maximum discrepancy a pixel in c can withstand. 

If we want to represent all images that are like c but there is a small region that has a very different brightness, due to a spurious dot in the image, then we should not use maximum norm, because it bounds the brightness difference for all pixels, but not some pixels. The brightness difference that results in the white dot is extreme, from 0 (black) to 1 (white). That is why instead we use the Euclidean norm. 

# Constraint-Based Verification

The idea behind constraint-based verification approaches is to encode the neural network and the properties we want to verify about it as one huge formula using first-order logic with linear real arithmetic. The validity of such formula implies that the neural network fulfills the desired properties. We can check for validity using SMT solvers, but we might need to modify those solvers to be more efficient for neural network verification. 

## First-Order Logic and Linear Real Arithmetic

First-order logic (FOL), also known as first-order predicate calculus, is a formal system used to express statements in a precise and unambiguous way. It is a mathematical language that allows us to represent complex relationships between objects, and to reason about them logically.

FOL is composed of a set of symbols that can be used to construct logical statements. These symbols include variables, constants, logical operators such as conjunction ($\land$), disjunction ($\lor$), negation ($\neg$), and implication ($\Rightarrow$), and quantifiers such as the universal ($\forall$) and existential quantifiers ($\exists$).

FOL is widely used in computer science, artificial intelligence, mathematics, and philosophy. It provides a foundation for reasoning about complex systems and can be used to encode knowledge and infer new facts.

The theory of linear real arithmetic (LRA) is an extension of FOL that adds a set of axioms and rules for reasoning about arithmetic involving real numbers. In LRA, statements can be made about the relationships between real numbers, such as inequalities and equations, and logical deductions can be made using the axioms of LRA.

LRA extends FOL by adding the following symbols: real numbers, addition, subtraction, multiplication, division, and inequalities. The axioms of LRA provide rules for reasoning about arithmetic with real numbers, and the resulting theory is capable of expressing complex mathematical relationships involving real numbers.

## Encoding Neural Networks

To encode a neural network in FOL with LRA, we will first start encoding each node $v$ separately as a formula $F_v$ that relates the output $v^o$ of the node with it's inputs $v^{in, 1}$, $v^{in, 2}$ a.s.o. As an example, we can formalize a node that computes the ReLU function in the following way:

![RELU](posts/formal-dnn/images/relu.png)

Next, we need to encode the edges between the nodes. This is straight forward as we just need to set each output variable of a node equal to the input variable of the node the corresponding edge is pointing to.
Then, we build the conjunction overall node formulas and edge formulas, which gives us the encoding of the entire network $F_G$. In this way, we cannot encode non-linear activation functions directly. The only way we can handle them is through over approximation.

Lastly, we have to include the correctness property. The final formula will have the following form:

![DNN Encoding](posts/formal-dnn/images/correctnessproperty.png)

## Neural Network Solvers

SAT solvers are algorithms used to solve the Boolean satisfiability problem (SAT), which is a fundamental problem in computer science and mathematical logic. The SAT problem asks whether a given Boolean formula can be satisfied by assigning Boolean values (true/false) to its variables. We can use them to check the formula encoding the neural network and its desired properties for its validity. Popular SAT solvers include Z3 by Microsoft and CVC4.

![DPLL algorithm](posts/formal-dnn/images/dpll.png)
<p align = "center">
Fig.3 - DPLL algorithm
</p>

The DPLL algorithm (see Figure 3) is a complete algorithm for solving the SAT problem. It stands for "Davis-Putnam-Logemann-Loveland" and was first described in 1962 by Martin Davis, Hilary Putnam, George Logemann, and Donald Loveland. The DPLL algorithm works by repeatedly applying two basic operations: deduction and search.

Deduction is the process of applying logical rules to simplify the original formula. Boolean constant propagation is a technique used by SAT solvers to simplify the formula by replacing some literals with their constant value. It involves identifying clauses that contain only one literal and assigning that literal a truth value accordingly. If a literal is known to be true (or false), then all clauses containing that literal can be simplified by removing the literal (or the negation of the literal) from the clause. This can lead to significant reductions in the size of the formula and can make it easier to solve.

Search is the process of exploring different assignments of truth values to variables. The DPLL algorithm uses a backtracking search, which means that it tries one assignment after another and backtracks when it encounters a conflict (i.e., a clause that cannot be satisfied under the current assignment).

![DPLL modulo theories](posts/formal-dnn/images/dpllt.png)
<p align = "center">
Fig.4 - DPLL modulo theories
</p>

DPLL modulo theories, $DPLL^T$ for short (see Figure 4), is an extension of the DPLL algorithm that can handle formulas with constraints over more complex domains, such as formulas in linear real arithmetic. First, it substitutes the constraints in linear arithmetic by simple Boolean variables, creating a Boolean abstraction $F^B$ of the formula that can be checked for validity by standard SAT solvers as discussed previously. In case $F^B$ is unsatisfiable, there is no way the original formula can be satisfiable, and we can return UNSAT. If is is satisfiable, DPLL will return a valid assignment of the variables $I$, called model of the formula. We can then compute $I^T$, an assignment for the original formula $F$ by substituting the Boolean variables back for their constraints in real arithmetic. To check if those constraints are consistent or not, we need a special theory solver, which we will present next. If they are, we found a model for $F$, if not we will set $F^B = F^B \lor \neg I$ to avoid that the DPLL algorithm will choose this inconsistent model in the next run of the algorithm. More details about the algorithms underlying modern SAT solvers can be found in the book [Handbook of Satisfiability](https://www.iospress.com/catalog/books/handbook-of-satisfiability-2).

As mentioned, the $DPLL^T$ algorithm needs access to a theory solver that takes in a conjunction of linear equalities and checks their satisfiablity. One such algorithm is the Simplex algorithm. 

The simplex algorithm is a widely used algorithm for solving linear programming problems, which are optimization problems where the objective function and the constraints are all linear functions. In verification, our interest is typically in finding any satisfying assignment, so we do not have an objective function to optimize. Instead of focusing on the mathematical details of the algorithm, we will present an illustrative example instead.

![Simplex example](posts/formal-dnn/images/simplex.png)
<p align = "center">
Fig.5 - Simplex example
</p>

Let's say we have the following linear constraints:

- $x + y \geq 0$
- $-2x + y \geq 2$
- $-10x + y \geq -5$


The constraints are visualized in Figure 5 and the area marked red is representing satisfying examples of x and y. We will begin with the initial interpretation that sets both x and y to 0, which is not a model of the formula, since constraint 2 is violated. We can decrease x to -1, but then we notice that constraint 1 is violated. To fix this, we can increase y to 2/3, a.s.o. until we arrive at a satisfying assignment, x = -2/3, y = 2/3. How to exactly adapt which variables and how to ensure termination of the algorithm is a detail we don't cover here, but can be read in detail in the corresponding paper.

To increase the performance of the outlined approach for neural network verification and make the solution scale to larger networks, Katz et al. proposed an extension to the Simplex algorithm, called [Reluplex](https://arxiv.org/pdf/1702.01135.pdf&xid=25657,15700023,15700124,15700149,15700186,15700191,15700201,15700237,15700242.pdf). The problem with the previous aproach is that piece-wise linear activation functions like ReLU will be encoded  as disjunctions, increasing the number of cases the DPLL algorithm needs to handle exponentially in the number of ReLUs. To fix this issue, Reluplex natively handles the ReLU constraints within Simplex in addition to the linear constraints. Their work eventually results in a state of the art SMT-based neural network verification solver, called [Marabou](https://github.com/NeuralNetworkVerification/Marabou).

# Abstract Interpretation-Based Verification

Despite performant algorithms, the approach of encoding a neural network along with a correctness property in first-order logic does not scale to large neural networks, because satisfiability modular linear real arithmetic is a NP-complete problem. Therefore, in this chapter, the focus will be on approximate techniques based on Abstract Interpretation. These techniques overapproximate the semantics of a neural network, so that they can proof correctness, but if they fail, we cannot say if the correctness property holds or not. 

## Abstract Interpretation

[Abstract interpretation](https://dl.acm.org/doi/pdf/10.1145/234528.234740) is a formal method for analyzing the behavior of computer programs. It provides a way to reason about the possible inputs and outputs of a program, as well as the potential errors or bugs that may arise during its execution.

The basic idea behind abstract interpretation is to create an abstract model of the program that captures its key properties, while ignoring details that are not relevant to the analysis. This model is then used to reason about the behavior of the program, using mathematical techniques to prove certain properties or detect potential issues.

Suppose we want to analyze a program that takes an input number and returns its square. The abstract model might represent the input as a range of possible values, rather than a single number. This allows us to reason about the behavior of the program for any input in that range, rather than just a specific input. It is impossible to test the program individually for all possible inputs, but Abstract Interpretation allows us to show correctness of the program for the whole range of inputs at once.

![Abstract interpretation](posts/formal-dnn/images/abstint.png)
<p align = "center">
Fig.6 - Abstract interpretation
</p>

As a visualization, consider above image. We have a very large number of possible program trajectories and a red area called the forbidden zone, that represents the values of the program that do not fulfill the correctness property. Instead of testing individual trajectories, we build an abstraction of them, represented as the green area. If we can show that the whole green area lies outside of the red area, we have proven correctness. The top diagram shows a legitimate error, while the bottom diagram shows an error that is due to overapproximation and does not represent an actual behavior of the program, a so called false alarm. 

Abstract interpretation has applications in a variety of areas, including software engineering, program optimization, and formal verification. It is particularly useful in contexts where the behavior of a program needs to be analyzed automatically, such as in static analysis tools or automated testing frameworks. A detailed course about Abstract Interpretation from beginner to advanced learner is [Principles of Abstract Interpretation](https://mitpress.mit.edu/9780262044905/principles-of-abstract-interpretation/) by the inventor of the method, Patrick Cousout.
 
## Abstractly Interpreting Neural Networks

Now, to apply Abstract Interpretation to the verification of neural networks, we first have to choose an appropriate abstract domain. Than we have to define so called abstract transformers, i.e., defining the operations of the neural network like addition, multiplication and compositions thereof in the abstract domain. Then we use this to abstractly interpret the whole network from start to finish on a set of inputs. Lastly, we need to check if the resulting set of outputs all fulfill the desired correctness property. But as already mentioned, since abstract interpretation is an approximate technique, if some outputs don't fulfill the property in the abstract domain, this does not necessarily mean that the property is not fulfilled in the general case. It could be just that the abstract domain we chose is too coarse-grained and overapproximates the behavior of the neural network too much.

For illustrative purposes, we will choose a simple abstract domain, the interval domain. An interval is defined by a lower and upper bound, $[l, u]$. If we want to add two intervals together, we will just add their lower and upper bounds respectively, $[l + l', u + u']$. Multiplication is trickier, because we have to take into consideration that the signs might flip. Therefore, the resulting interval is $[min(B), max(B)]$, where $B = \{l * l', l * u', u * l', u * u'\}$ contains all possible combinations of lower and upper bounds of the two intervals.

We can use the principles for multiplication and addition to compute any affine function on intervals. What is missing are the non-linear activation functions like ReLU that are used in neural networks. It turns out that most of them are actually monotonically increasing. It is easy to define an abstract transformer for a monotonic function $f$. We just have to apply the function on the lower and upper bounds of the interval to yield $[f(l), f(u)]$.

![Interval, Zonotope and Polyhedra abstraction for ReLU](posts/formal-dnn/images/domains.png)
<p align = "center">
Fig.7 - Interval, Zonotope and Polyhedra abstraction for ReLU
</p>

Now to abstractly interpret a whole neural network, we first define the range of possible inputs for each input node in the form of an interval. At each node, we use the abstract transformer of its function to produce intervals as outputs and propagate them through the network. The interval domain can be computed very efficiently, but it overapproximates the network a lot. The biggest problem is that the interval domain is non-relational. It cannot capture relations between different dimensions. More precise abstractions are needed, but as a trade-off, they will be less efficient. Examples for this include zonotopes and polyhedra. You can look at Figure 7 for a comparison of the preciseness of those domains in approximating the ReLU function. Regardless of the chosen domain, the general principle remains the same, we just have to define abstract transformers for affine functions and non-linear activation functions of the neural network.
 
The [ETH Robustness Analyzer for Neural Networks (ERAN)](https://github.com/eth-sri/eran)  is an advanced and versatile analyzer that utilizes abstract interpretation for precise, efficient, and scalable verification of neural networks. It is developed by the SRI Lab at ETH Zurich, ERAN is a part of the Safe AI project. It can automatically verifying DNNs with feedforward, convolutional, and residual layers against various types of input perturbations, including L∞-norm attacks, geometric transformations, and vector field deformations. 

# Conclusion

We have seen two neural network verification approaches, one complete one based on SMT solvers and the other, an incomplete approach, based on abstract interpretation. There exist of course also other approaches beyond those two categories. To verify realistic safety-critical deep learning based systems, there are still challenges along the way.
How to come up the right effective properties to verify challenging, domain-dependent problems? How to keep up in scalability of the verification approaches in face of deep learning networks that rapidly grow in their complexity? And lastly, how can we cope with a dynamic environment and reinforcement learning?

Further resources on the topic of neural network verification include [Introduction to Neural Network Verification](https://arxiv.org/pdf/2109.10317.pdf) and [Algorithms for Verifying Deep Neural Networks](https://arxiv.org/pdf/1903.06758.pdf). To apply those advanced techniques to real world problems and improve the state-of-the-art, a deep knowledge in computer vision, deep neural networks, formal methods and mathematical optimization is necessary. 

Every year since 2020, the [International Verification of Neural Networks Competition (VNN)](https://sites.google.com/view/vnn2022) is hosted and that crowns the most efficient solver. The winner of 2022 was Alpha-Beta CROWN. The results are promising and give hope that these approaches can be used to verify complex systems like autonomous driving components in the future.