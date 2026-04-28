---
layout: post
title: "Software Engineering as Scientific Infrastructure"
author: Matthias Kretz
tags: [software engineering, software architecture, standardization, maintenance]
---

A "brain dump" …

## Problem “Statement”

* Science in many cases relies heavily on software.
* Software needs to be performance portable to newer hardware and vendor independent.
* Science software requires stable and long-lived dependencies. Being able to rely on actual standards (ISO, DIN, ECMA, etc.) is helpful.
* Custom software solutions might generalize to different science projects but:
  * The effort to provide a supported and maintained project can't easily be financed.
  * Filling a (permanent) software engineer position with capable talent is challenging in the presence of well paying commercial competition.
  * Learning about useful software solutions from other science projects is often only a matter of luck.
  * Acquiring the necessary know-how to integrate a foreign software solution is challenging.
  * The effort to standardize a solution is impossible to manage when the main focus of the people involved is a scientific career.
* Computer science research groups are not a good fit for building reliable, tested, and long-term supported software projects.
  * There is little potential for writing impactful research papers when working on the 80% effort required to get the last 20% of a solution implemented and stabilized.
  * The long-term maintenance and user support has little to no scientific aspects.

## Vision

Software engineering, software project maintenance, standardization of solutions, and consulting services to science groups requiring custom software solutions needs to be recognized as an integral *infrastructure* of (European) science. Recognizing software as infrastructure is important to make scientific software development independent from the need to research novel solutions and instead focus on stabilizing and maintaining such solutions when they come out of research groups.

### Software engineering

* Software engineering is different from computer science in that it focuses on long-lasting architecture designs, quality, and reusability. Computer science in principle often does not have to move software projects further than the proof-of-concept stage.
* Physics projects (as one example of many disciplines) typically depend on physicists to learn just enough about programming that they can extend an existing application to support their area of research. This has profound implications on code quality.
* While there certainly are highly capable software developers in all science branches, this cannot be the expectation.
* A good mix of domain specialists and software engineers can improve quality, reliability, efficiency, and hardware independence, with potentially an increased productivity on the science side.

### Software project maintenance

* Maintaining software is work, sometimes a lot of work.
* The environment in which software is built and executed is constantly and rapidly changing.
* Programming languages are evolving (e.g. the latest FORTRAN standard is from 2018, C++ from 2020).
* Especially shared software components require constant maintenance effort.
* Users find bugs and request features.
* Failure to act on user feedback implies users need to look for a different solution, potentially defeating the purpose of sharing the software component in the first place.
* Maintenance requires the reliability of well-paid software engineer positions that back this infrastructure other science projects rely on.

### Standardization

* *TODO*

### Consulting

* This may actually be the most important part.
* It is not enough to have a web page, mailing list / forum, etc.
* Discussing software architecture and design requires face to face (potentially via video conference) interaction.
* Science software projects should be able to collaborate with experienced software engineers on their code base.
* Supporting science software projects to use an external dependency with active participation in design and development facilitates increased collaboration and trust. It is also highly valuable experience for further research and development on shared software resources.
* These interactions are important to help identify new projects that the wider science community can benefit from, that can either be consolidated with existing efforts or incorporated into the body of shared software components.

## Organization

* We need a large (and probably distributed) group of software engineers that can work on the complete stack relevant for software development: from compiler engineers to library experts, from low-level optimization to high level abstraction expertise.
  * *TODO*: How can we build such a group?
* We need permanent positions of highly qualified software engineering experts to produce the quality required to support the massive software projects our science domain experts require for their research.
* This needs to be a service to all our science and is as much a permanent infrastructure as a particle accelerator and its staff is for high energy physics research.
