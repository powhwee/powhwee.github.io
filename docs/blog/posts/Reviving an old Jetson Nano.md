---
date:
  created: 2024-06-15
  updated: 2024-06-16
title: "Reviving and old Jetson Nano"
categories:
  - Technology
  - Web Development
tags:
  - under construction

draft: false
---
<!-- more -->



# Summary of Learnings: AI Model Deployment on NVIDIA Jetson Nano

This document summarizes the key technical learnings from the process of deploying a TensorFlow-based object detection model on an NVIDIA Jetson Nano. The process involved troubleshooting a series of dependency, compilation, and model compatibility issues.

### 1. The Jetson Nano Software Environment

The process highlighted the characteristics of the Jetson Nano's software environment and its constraints.

*   **Inference-Optimized Architecture:** The hardware is designed for running pre-trained models (inference), which explains why compiling libraries from source is a slow and memory-intensive process.
*   **Toolchain Versioning:** The JetPack SDK is based on older, stable versions of tools (e.g., Python 3.6, g++ 7.x, and cmake 3.10). This creates compatibility conflicts when attempting to install modern AI libraries that have newer requirements.
*   **AArch64 Architecture:** The `aarch64` (ARM64) architecture requires specifically compiled software packages. This was the cause of the `Illegal instruction` error and necessitated the use of official NVIDIA-provided wheels for TensorFlow.

### 2. Dependency Management and Problem Solving

A series of dependency issues were encountered and solved, establishing a troubleshooting methodology.

*   **Problem: Python Package Incompatibility.**
    *   **Observation:** The latest versions of packages on the Python Package Index (PyPI) are often not compatible with the Jetson's default Python 3.6.
    *   **Examples Solved:** `protobuf` requiring Python 3.7+, `onnx-simplifier` requiring a newer `onnxruntime`.
    *   **Solution:** Manually installing older, specific package versions (e.g., `pip install onnx==1.11.0`).

*   **Problem: Missing System-Level Libraries.**
    *   **Observation:** Python packages with C++ backends often depend on system libraries that must be installed with `apt-get`.
    *   **Examples Solved:** `h5py` failing to build without `libhdf5-dev`, `protobuf` failing to build without `protobuf-compiler`.
    *   **Solution:** Reading build logs to identify missing files and installing the corresponding `-dev` package.

*   **Problem: Build Tool Incompatibility.**
    *   **Observation:** The tools used to build packages, such as `Cython` and `cmake`, also have version dependencies.
    *   **Examples Solved:** `numpy` failing to build because the default `Cython` was too new; `onnx-simplifier` failing because the system `cmake` was too old.
    *   **Solution:** Manually installing a specific version of the build tool (e.g., `pip install Cython==0.29.36`) or upgrading the system tool itself.

### 3. The Model Conversion Pipeline

The conversation detailed the multi-stage process of preparing a model for the Jetson platform.

*   **The Workflow:** The required workflow is **TensorFlow -> ONNX -> TensorRT**.
*   **The Protobuf Performance Requirement:** The `protobuf` library's C++ accelerated backend is necessary for acceptable performance during model conversion. The solution involved building the library from source to create and install this backend.
*   **Required Model Patching:** The ONNX model generated from TensorFlow was not directly compatible with TensorRT and required two modifications via custom Python scripts:
    1.  Changing the model's **input data type** from `UINT8` to `FP32`.
    2.  Changing the model's **internal parameters and graph definitions** from `INT64` to `INT32`.
*   **TensorRT Engine Portability:** The final `.engine` file is compiled for a specific device and software version. This is why the original pre-built engine failed and why a new one had to be generated on the device.

### 4. Troubleshooting Principles

Several troubleshooting techniques were used to resolve the issues.

*   **Error Log Analysis:** Each solution was derived from reading the error logs to find the root cause, from `FileNotFoundError` to `TypeError: does not support assignment`.
*   **Virtual Environment Behavior:** The process highlighted how virtual environments function, specifically that `sudo pip install` targets the system's Python, not the active environment, and that file permissions can get mixed between the user and root.
*   **Clean Build Environment:** When permissions became tangled, the solution was to delete the source directory (`sudo rm -rf protobuf`) and start the build process from a clean state.
*   **API-Specific Object Manipulation:** It was discovered that the `RepeatedCompositeContainer` object in the ONNX library, while list-like, does not support methods like `.clear()` or slice assignment (`[:] =`). The correct manipulation required iterating with `del` and then using the `.extend()` method.