---
layout: post
title:  "Trakkt - Dev Blog #1 - RAG"
category: Projects
---

# Overview
Trakkt is a project I've been working on in the background for a while now. Whilst it's under wraps on what it does currently, I can share some unique insights into the development process, and the technologies I'm using to build it.

The first major hurdle I have is to outline the templates for a machine. Excavators, bulldozers, cranes, earth movers, and more. Each machine has its own unique set of data points, service intervals, maintenance requirements, and more. I have a lot (and I mean a lot) of unstructured, and public data on these machines (hundreds of megabytes for each machine) that I need to structure in a standard format, so I can use it to power the app. Welcome to the world of RAG (Retrieval Augmented Generation).

# Prototype
The initial prototype of the templating system involved locally ingesting the PDF data into QDrant (Using OCR where necessary), and then using a Phi model to query the data, and generate a structured JSON output. This actually worked quite well: Templates generated, mostly correct data and a good starting point for the next phase of development.

