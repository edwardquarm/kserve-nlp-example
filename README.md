# KServe NLP Example

This repository demonstrates how to deploy a pre-trained text classification model using [KServe](https://kserve.github.io/). The model and tokenizer are downloaded from Huggingface Transformers.

---

## Overview

The goal of this project is to:
1. Download a pre-trained tokenizer and classifier model for text classification.
2. Deploy the model to KServe for scalable inference.

---

## Steps to Use This Repository

### Prerequisite
Install the required dependencies:
```bash
pip install -r requirements.txt
```

---

### 1. Download the Model and Tokenizer

#### Configure `model-config.yaml`
Set the following parameters in `model-config.yaml`:

- **`model_name`**: Pre-trained models like `bert-base-uncased`, `roberta-base`, etc.
- **`mode`**: Choose from `sequence_classification`, `token_classification`, `question_answering`, or `text_generation`.
- **`do_lower_case`**: `true` or `false` to configure the tokenizer.
- **`num_labels`**: Number of outputs (e.g., `2` for sequence classification).
- **`save_mode`**: `"torchscript"` or `"pretrained"`.
- **`max_length`**: Maximum input sequence length.
- **`captum_explanation`**: `true` for eager mode models; `false` for TorchScript models.
- **`embedding_name`**: Embedding layer name (e.g., `bert` for `bert-base-uncased`).
- **`hardware`**: Target platform (`neuron` or `neuronx`).
- **`batch_size`**: Batch size for tracing the model.

#### Download the Model
Run the following command to download the model and tokenizer:
```bash
python Download_Transformer_models.py
```
This will create the required files in the `Transformer_model` directory.

---

### 2. Prepare the Model for TorchServe

#### Directory Structure
Organize your files as follows:
```
├── Download_Transformer_models.py       # Script to download the tokenizer and model
├── Transformer_handler_generalized.py   # Custom handler for TorchServe
├── Transformer_model                    # Directory containing the downloaded model files
│   ├── model.safetensors                # Serialized model file
│   ├── config.json                      # Model configuration file
├── text_classification_artifacts        # Artifacts for text classification
│   ├── index_to_name.json               # Maps predictions to labels
│   ├── sample1.txt                      # Sample input text for inference
├── config
│   ├── config.properties                # TorchServe configuration settings
├── model-store
│   ├── text_classification.mar          # Model archive file created using torch-model-archiver
```

- **`Download_Transformer_models.py`**: Script to download the tokenizer and model.
- **`Transformer_handler_generalized.py`**: Custom handler for TorchServe.
- **`Transformer_model/`**: Directory containing the downloaded model files.
- **`text_classification_artifacts/`**: Artifacts for text classification.
- **`config/`**: Contains TorchServe configuration settings.
- **`model-store/`**: Contains the `.mar` file created using `torch-model-archiver`.

#### Create the `.mar` File
Use the following command to create the `.mar` file:
```bash
torch-model-archiver --model-name DISTLBERTClassification \
                     --version 1.0 \
                     --serialized-file Transformer_model/model.safetensors \
                     --handler Transformer_handler_generalized.py \
                     --config-file model-config.yaml \
                     --extra-files "Transformer_model/config.json,text_classification_artifacts/index_to_name.json"
```

---

### 3. Register the Model with TorchServe

Register the model using the following commands:
```bash
mkdir model-store
mv DISTLBERTClassification.mar model-store/
torchserve --start --model-store model-store --models my_tc=DISTLBERTClassification.mar --ncs --disable-token-auth --enable-model-api
```

---

### 4. Run an Inference Locally

To test the model locally, run:
```bash
curl -X POST http://127.0.0.1:8080/predictions/my_tc -T Huggingface_Transformers/text_classification_artifacts/sample1.txt
```

---

### 5. Deploy to KServe

#### Update the KServe YAML Configuration
Use the following `ServingRuntime` configuration to deploy the model:
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
Update the paths to point to your model storage location.

---

### 6. Test the Deployment

Send test requests to the deployed model endpoint to verify its functionality.

---

## Repository Structure

- **`Download_Transformer_models.py`**: Script to download the tokenizer and model.
- **`model-config.yaml`**: Configuration file for setting up the model parameters.
- **`Transformer_handler_generalized.py`**: Custom handler for TorchServe.
- **`config/`**: Contains TorchServe configuration settings (`config.properties`).
- **`model-store/`**: Contains the `.mar` file created using `torch-model-archiver`.
- **`README.md`**: This file, providing an overview of the project.

---

## References

- [Huggingface Transformers](https://huggingface.co/transformers/)
- [KServe Documentation](https://kserve.github.io/)

---

## License

This project is licensed under the same terms as the Huggingface Transformers library. Please refer to the [Huggingface Transformers repository](https://github.com/huggingface/transformers) for more details.
