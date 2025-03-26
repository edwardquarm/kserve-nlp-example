# KServe NLP Example

This repository demonstrates how to deploy a pre-trained text classification model using [KServe](https://kserve.github.io/). The model and tokenizer are downloaded from Huggingface Transformers.

## Overview

The goal of this project is to:
1. Download a pre-trained tokenizer and classifier model for text classification.
2. Deploy the model to KServe for scalable inference.

## Steps to Use This Repository

### 1. Download the Model and Tokenizer
- Use the `Download_Transformer_models.py` script located in the `Huggingface_Transformer` folder to download the pre-trained tokenizer and classifier model.  

### 2. Prepare the Model for TorchServe
- Store your model and dependent files in the following structure:
  ```
  ├── config
  │   ├── config.properties
  ├── model-store
  │   ├── text_classification.mar
  ```
  - The `config.properties` file should contain TorchServe configuration settings.
  - The `.mar` file is the model archive file created using the `torch-model-archiver` tool.

- Example command to create the `.mar` file:
  ```bash
  torch-model-archiver --model-name text_classification \
                       --version 1.0 \
                       --serialized-file ./model/pytorch_model.bin \
                       --handler ./handler.py \
                       --extra-files "./model/config.json,./model/tokenizer.json" \
                       --export-path ./model-store
  ```

### 3. Deploy to KServe
- Use the provided deployment template to deploy the downloaded model to KServe.
- Update the KServe YAML configuration file with the appropriate model path and container details.
- Use the following `ServingRuntime` configuration to deploy the model using TorchServe on KServe:
  ```yaml
  apiVersion: serving.kserve.io/v1alpha1
  kind: ServingRuntime
  metadata:
    name: kserve-torchserve
  spec:
    annotations:
      prometheus.kserve.io/port: "8082"
      prometheus.kserve.io/path: /metrics
    supportedModelFormats:
      - name: pytorch
        version: "1"
        autoSelect: true
        priority: 2
    protocolVersions:
      - v1
      - v2
      - grpc-v2
    containers:
      - name: kserve-container
        image: pytorch/torchserve-kfs:0.9.0
        args:
          - torchserve
          - --start
          - --model-store=/mnt/models/model-store
          - --ts-config=/mnt/models/config/config.properties
        resources:
          requests:
            cpu: "1"
            memory: 2Gi
          limits:
            cpu: "1"
            memory: 2Gi
        ports:
          - containerPort: 8080
            protocol: TCP
  ```

- Update the paths in the configuration to point to your model storage location.

### 4. Test the Deployment
- Send test requests to the deployed model endpoint to verify its functionality.

## Repository Structure
- **`Huggingface_Transformer/`**: Contains the `Download_Transformer_models.py` script for downloading the tokenizer and model.
- **`deployment/`**: Includes templates and configuration files for deploying the model to KServe.
- **`README.md`**: This file, providing an overview of the project.

## References
- [Huggingface Transformers](https://huggingface.co/transformers/)
- [KServe Documentation](https://kserve.github.io/)

## License
This project is licensed under the same terms as the Huggingface Transformers library. Please refer to the [Huggingface Transformers repository](https://github.com/huggingface/transformers) for more details.
