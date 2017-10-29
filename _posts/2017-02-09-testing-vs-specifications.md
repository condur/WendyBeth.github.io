---
layout: post
title: "Testing vs Specifications"
description: "Testing as an approach to understand the system is not effective, a potential solution is to use specifications."
tags: [management]
---

I recently read again the paper “Out of the Tar Pit” by Ben Moseley and Pater Marcks. The paper is about how to manage complexity in large-scale software systems. This is a common issue with building and maintaining not just large software systems, the authors are proposing a Functional Reactive Programming (FRP) approach for hand complexity. I would like to stop in this article on a specific topic that is covered in the paper: __testing - as one approach to understanding the systems__.

Testing is attempting to understand a system from the outside — as a “black box”. Conclusions about the system are drawn on the basis of observations about how it behaves in certain specific situations. Testing may be performed either by human or by machine. The former is more common for whole-system testing, the latter more common for individual component testing.

The key problem with testing is that a test (of any kind) that uses one particular set of inputs tells you nothing at all about the behavior of the system or component when it is given a different set of inputs. The huge number of different possible inputs usually rules out the possibility of testing them all, hence the unavoidable concern with testing will always be — have you performed the right tests? The only certain answer you will ever get to this question is an answer in the negative — when the system breaks.

A quote from Edsger W. Dijkstra about testing from "On the reliability of programs":
>“testing is hopelessly inadequate....(it) can be used very effectively to show the presence of bugs but never to show their absence.”

This is not to say that testing has no use. The bottom line is that all ways of attempting to understand a system have their limitations (and this includes both informal reasoning — which is limited in scope, imprecise and hence prone to error — as well as formal reasoning — which is dependent upon the accuracy of a specification). Because of these limitations, it may often be prudent to employ both testing and reasoning together.

Given a stark choice between investment in testing and investment in simplicity, the latter may often be the better choice because it will facilitate all future attempts to understand the system — attempts of any kind.

The authors recommend investing in simplicity rather testing which I found hard to sell to the owner of the product and even to the software engineering team. So what is the solution? Even I don't have practical experience with the solution that I want to propose, I strongly believe that is definitely worth trying.

A potential solution is a library in Clojure community that is in active development right now: clojure.spec. The library is trying to unify the way how data and function specifications are defined and how can be used for error reporting, destructuring, validation, generative testing, better communication, improved developer experience and overall to have a more robust software. I encourage you to read the spec rationale and overview here: [clojure.org/about/spec](https://clojure.org/about/spec).

The more I think I about this library the more places I found where can be used and prevent software systems expensive mistakes and maintenance costs.
