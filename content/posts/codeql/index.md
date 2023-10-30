---
title: "Next Generation Static Analysis: CodeQL"
description: "The stages of a secure software development lifecycle and how CodeQL can be integrated to build more secure software"
tags: [devsecops, sast]
date: 2023-07-21
math: false
---

This article gives an introduction to CodeQL and how to use it to improve code security via query-based code inspection. The tool can be used to find vulnerabilities and enables custom security check queries to help find problems so code can be more readily improved. The queries run relatively fast and are formatted in a similar way to database SQL queries, making it easy to use. I will end with a small demonstration of CodeQL in action.

# Secure Software Development Lifecycle

CodeQL can be part of a larger secure software development lifecycle (SSDLC), with the goal to integrate security testing into every stage of the development process.

![SSDLC](posts/codeql/images/ssdlc.png)

<p align = "center">
Fig.1 - The stages of a secure software development lifecycle
</p>

We can divide this process into 5 different stages: Develop, Inherit, Build, Deploy and Operate. At each stage, different security testing methodologies can come into play, as can be seen in Figure 1.

Threat modelling involves identifying and analyzing potential security risks early on, allowing developers to proactively address these issues during the development stage. Development standards, such as coding guidelines and best practices, help maintain consistency and security throughout the codebase. Static Application Security Testing (SAST) tools, like CodeQL, on the other hand, scan the source code for known security flaws and coding errors, enabling early detection and remediation of vulnerabilities.

In the Inherit stage, the focus shifts to managing software dependencies effectively. Software Composition Analysis (SCA) tools are a key component of this phase. They are used to identify and assess the third-party libraries that a project relies on. By thoroughly analyzing these dependencies, development teams can ensure that they meet security and compliance standards. Clear policies for dependency management and the creation of an inventory of all utilized component are paramount to prevent vulnerabilities arising from outdated or insecure dependencies.

The Build stage is where we test the final product as a whole. Dynamic Application Security Testing (DAST) tests the software during runtime, simulating real-world attacks. Compliance Testing ensures that the software complies with industry standards, legal regulations, etc. while infrastructure testing assesses the security of the underlying infrastructure where the application will be deployed. This includes examining servers, databases, and network configurations for vulnerabilities and misconfigurations that could be exploited.

The Deploy stage should be as automated as possible to avoid human error, e.g. through Infrastructure as Code tools like Terraform.

The Operate stage is where the software runs in production. To ensure the security of the application we can make use of Cloud Security Monitoring, Runtime Application Self-Protection, and Bug Bounties.
Cloud Security Monitoring involves continuous surveillance and analysis of the cloud infrastructure. This is essential for identifying and responding to potential security incidents, abnormal behaviour or vulnerabilities in a cloud-based environment. RASP is an additional security layer embedded within the application itself. It dynamically monitors the application's behavior during runtime, identifying and mitigating potential security threats and attacks in real-time. Bug Bounties on the other hand, are programs that invite external security researches (bug hunters) to identify and report vulnerabilities in the software. By incentivizing ethical hacking, organizations can discover and address security issues that might otherwise go unnoticed.

# What is CodeQL and what can we do with it?

CodeQL is a declarative static analysis tool. It creates a database of facts from a program source, then runs queries over those facts to extract information (see Figure 2).

![CodeQL Components](posts/codeql/images/components.png)

<p align = "center">
Fig.2 - CodeQL components
</p>

Queries can be used to find bugs and security vulnerabilities in large codebases through automated scans. In contrast to older generations of SAST tools which struggle with false positives, with CodeQL, by adjusting the query, the analysis can be made more precise easily. Lastly, the queries represent codified, readable and executable security knowledge that can be shared across teams.

# How does a query look like?

In Figure 3, you can see a simple query and its results, detecting empty Else-Blocks. The query is easy to understand and looks similar to a standard SQL query. The code is language specific, but largely similar across languages.

![Example Query](posts/codeql/images/simple-query.png)

<p align = "center">
Fig.3 - Example Query
</p>

To model more complex queries, CodeQL provides other language constructs, namely predicates and classes. In Figure 4, we rewrite the same original query using predicates and classes, respectively.

![Predicates and Classes](posts/codeql/images/predicates-classes.png)

<p align = "center">
Fig.4 - Predicates and Classes
</p>

# Advanced Functionality

CodeQL also offers advanced analysis modes. Variant Analysis takes a known vulnerability and models its characteristics into a CodeQL query, which can then be run across several repositories to find vulnerabilities.
Taint Tracking Analysis emulates a program run and tracks data from an origin, called the source, usually user provided input, to a destination, called sink representing a vulnerable or dangerous function. If there is a flow from source to sink, it means a vulnerability might exist in the code. To run a Taint Tracking Analysis, we need to define sources, sinks, and optionally taint steps and sanitizers. Next we will see a small example that uses taint tracking to detect a SQL injection vulnerability.

# Hands-On

The goal of this demonstration is to write a query to detect a simple SQL injection within [OWASP Juice Shop](https://github.com/juice-shop/juice-shop), dubbed the most modern and sophisticated insecure web application.

![SQL Injection](posts/codeql/images/sqli.png)

<p align = "center">
Fig.5 - SQL Injection
</p>

SQL Injection is a simple to understand vulnerability. An attacker can inject code to modify the execution of SQL queries to access sensitive data (see Figure 5). Tools like [sqlmap](https://sqlmap.org/) can automate the exploitation of those vulnerabilities to some extent. The vulnerability can be prevented by using [Prepared Statements](https://portswigger.net/web-security/sql-injection#how-to-prevent-sql-injection).

![Vulnerable Login](posts/codeql/images/exploitation.png)

<p align = "center">
Fig.6 - Vulnerable Login
</p>

In Figure 6, you can see the exploitation of a SQL injection vulnerability in the login functionality of the OWASP juice shop. Inside the email form input, we close the input string and append "--" to start a comment in SQLite syntax, causing the query to not check the password when querying for the user to log in.

To get started writing the query, we need to setup CodeQL. The easiest way to do this, is to install the CodeQL extension for VS Code and clone the Github repository containing the [starter workspace](https://github.com/github/vscode-codeql-starter) that contains CodeQL libraries and queries for all supported languages. This is where we can write queries for testing purposes. Using the extension, we add a CodeQL database. In our case, we just have to provide it with the Github repository link of OWASP Juice Shop and we are good to go!

Analyzing the code, we first identify the source and sink of the vulnerability (see Figure 7):

![Source and Sink](posts/codeql/images/source-sink.png)

<p align = "center">
Fig.7 - Source and Sink
</p>

The source, or user-controlled input, is the request object, containing email and password provided upon login in. The sink, or vulnerable function, is the models.sequelize.query function.

We will now write our own query to detect this and similar vulnerabilities in the code. We will make use of taint tracking to detect if there is a flow from source to sink. To do this, we have to fill in some boilerplate code, to specify what our sources and sinks are in the form of predicates (see Figure 8).

![Our own query](posts/codeql/images/own-query.png)

<p align = "center">
Fig.8 - Our own query
</p>

Of course, we would not need to write such a query by ourselves, since CodeQL already provides queries for all kinds of vulnerabilities, for example, [this](https://github.com/github/codeql/blob/8e890571ed7b21bc10698c5dbd032b9ed551d8f1/javascript/ql/src/Security/CWE-089/SqlInjection.ql) is the actual, much more complex, query used to detect SQL injections. Anyway, when we run our query, we get the following results:

![Query Results](posts/codeql/images/results.png)

<p align = "center">
Fig.9 - Query Results
</p>

It finds our login vulnerabity and two other SQL injection vulnerabilities of similar kind in the codebase! This concludes this short introduction to CodeQL.

# Similar tools

In following blog posts, we will have a closer look at similar SAST tools and how the differ to CodeQL, namely [semgrep](https://semgrep.dev/) and [Joern](https://docs.joern.io/).

# Sources

The best way to learn more about CodeQL is to read the [Github Blog Post Series](https://github.blog/2023-03-31-codeql-zero-to-hero-part-1-the-fundamentals-of-static-analysis-for-vulnerability-research/) and try out practical challenges, like the [Github Securitylab CTF](https://securitylab.github.com/ctf). Further Challenges are also linked in the above mentioned blog post series! If you want to see a more realistic SQL injection vulnerability and how to detect it with CodeQL, check out [this challenge](https://github.com/rvermeulen/codeql-workshop-cve-2021-21380).

For SSLDC and DevSecOps in general, the course [DevSecOps: Building a Secure Continuous Delivery Pipeline](https://www.linkedin.com/learning/devsecops-building-a-secure-continuous-delivery-pipeline) contained helpful information.
