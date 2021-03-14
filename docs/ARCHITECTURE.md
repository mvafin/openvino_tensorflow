# Architecture of Intel<sup>®</sup> OpenVINO™ Add-on for TensorFlow

This document describes the high-level architecture of Intel<sup>®</sup> OpenVINO™ Add-on for TensorFlow. This add-on is registered as a graph optimization pass in TensorFlow and optimizes the execution of supported operator clusters using Intel<sup>®</sup> OpenVINO™ runtime. Unsupported operators fall back to native TensorFlow runtime.

## Architecture Diagram

The below diagram shows the high level architecture of Intel<sup>®</sup> OpenVINO™ Add-on for TensorFlow. We broadly categorize the software stack into different modules as shown below. The purple box at the bottom shows the components of Intel<sup>®</sup> OpenVINO™ including different device plugins along with the corresponding libraries.

<p align="center">
  <img src="../images/openvino_tensorflow_architecture.png" width="450">
</p>

## Description of modules

In this section, we describe the functionality of each module and how it transforms the original TensorFlow graph.

#### Operator Capability Manager

Operator Capability Manager (OCM) implements several checks on TensorFlow operators to determine if they are supported by OpenVINO backends. The checks include supported operator types, data types, attribute values, input and output nodes, and many more conditions. The checks are implemented based on the results of several thousands of operator tests and model tests. OCM is continuously evolving as we add more operator tests and model tests to our testing infrastructure. This is an important module that determines which layers in the model should go to OpenVINO backends and which layers should fall back on native TensorFlow runtime. OCM takes TensorFlow graph as the input and returns a list of operators that can be marked for clustering so that the operators can be run on OpenVINO backends.

#### Graph Partitioner

Graph partitioner examines the nodes that are marked for clustering by OCM and performs further analysis on them. In this stage, the marked operators are first assigned to clusters. Some clusters are dropped after further analysis. For example, if the cluster size is very small or if the cluster is not supported by the backend after receiving more context, then the clusters are dropped and the operators fall back to native TensorFlow runtime. Each cluster of operators is then encapsulated into a custom operator that is executed on OpenVINO.

#### TensorFlow Importer

TensorFlow importer translates the TensorFlow operators in the clusters to OpenVINO nGraph operators with the latest available [operator set](https://docs.openvinotoolkit.org/latest/openvino_docs_ops_opset.html) for a give version of OpenVINO™ toolkit. An [nGraph function](https://docs.openvinotoolkit.org/latest/openvino_docs_nGraph_DG_build_function.html) is built for each of the clusters. Once created, it is wrapped into an OpenVINO CNNNetwork that holds the intermediate representation of the cluster to be executed on OpenVINO backend.

#### Backend Manager

Backend manager creates a backend to execute the CNNNetwork. We implemented two types of backends: basic backend and VAD-M backend. Basic backend is used for Intel CPUs, Intel integrated GPUs and Intel® MovidiusTM Vision Processing Units (VPUs). It creates an inference request and runs inference on a given input data. VAD-M backend is used for Intel® Vision accelerator Design with 8 Intel MovidiusTM MyriadX VPUs (referred as VAD-M or HDDL). We support batched inference execution in VAD-M backend. When a batched input is provided by the user, we create multiple inference requests and run inference in parallel on all the available VPUs in VAD-M.