# LLM-as-a-Judge Evaluation Setup

This directory contains the configuration for running automated evaluations using your deployed Llama model as both the subject and the judge. It's based on the MT-Bench evaluation framework but adapted to work with your OpenShift AI infrastructure.

## What This Does

The evaluation job will:
- Load a dataset of questions and reference answers from MT-Bench
- Generate responses using your deployed Llama 3.2 3B Instruct model
- Use the same model to judge the quality of those responses
- Calculate correlation metrics between the model's self-assessment and human ratings

## Configuration Overview

### Model Setup
- **Subject Model**: `llama-32-3b-instruct` (the model being evaluated)
- **Judge Model**: `llama-32-3b-instruct` (the same model judging itself)
- **Endpoint**: Your deployed model on OpenShift AI
- **Namespace**: `models` (where your model is deployed)

### Evaluation Task
The job uses the MT-Bench single-turn evaluation, which focuses on:
- Question-answer pairs from the first turn of conversations
- Rating responses on a 1-10 scale
- Measuring correlation with human judgments

### Dataset
- **Source**: `OfirArviv/mt_bench_single_score_gpt4_judgement` from HuggingFace
- **Filter**: Only first-turn conversations with empty reference answers
- **Purpose**: Tests the model's ability to provide helpful, accurate responses

## How to Deploy

1. **Make sure your model is running**:
   ```bash
   oc get inferenceservice llama-32-3b-instruct -n models
   ```

2. **Apply the evaluation job**:
   ```bash
   oc apply -f lmevaljob.yaml
   ```

3. **Monitor the job**:
   ```bash
   oc get lmevaljob custom-eval -n models
   oc logs -f job/custom-eval -n models
   ```

## What to Expect

The evaluation will:
- Take some time to complete (depends on dataset size and model speed)
- Generate logs showing progress and sample evaluations
- Produce metrics including Spearman correlation scores
- Create a final report with the model's self-assessment accuracy

## Customization

### Adjusting the Judge Model
If you want to use a different model for judging, modify the `inference_model` section in the YAML:

```yaml
"inference_model": {
    "__type__": "openai_completions",
    "model": "your-other-model",
    "base_url": "https://your-other-model-models.apps.your-cluster.com/v1/completions",
    "max_new_tokens": 256
}
```

### Changing the Dataset
To use a different evaluation dataset, update the `loader` section:

```yaml
"loader": {
    "__type__": "load_hf",
    "path": "your/dataset/name",
    "split": "train"
}
```

### Modifying the Rating Scale
The current setup uses a 1-10 rating scale. To change this, update the instruction template and output format in the `templates` section.

## Troubleshooting

### Common Issues

**Job fails to start**:
- Check that the model is running and accessible
- Verify the namespace matches where your model is deployed
- Ensure the model endpoint URL is correct

**Dataset download fails**:
- The dataset is public, so this shouldn't happen
- Check if your cluster has internet access to HuggingFace

**Low correlation scores**:
- This is expected for self-evaluation
- The model might be too lenient or harsh on itself
- Consider using a different model for judging

### Useful Commands

```bash
# Check job status
oc describe lmevaljob custom-eval -n models

# View job logs
oc logs -f job/custom-eval -n models

# Delete the job if needed
oc delete lmevaljob custom-eval -n models
```

## Understanding the Results

The evaluation produces several metrics:
- **Spearman Correlation**: How well the model's ratings correlate with human ratings
- **Rating Distribution**: How the model distributes its 1-10 ratings
- **Sample Evaluations**: Examples of the model's self-assessment

A high correlation score suggests the model has good self-awareness about response quality. A low score might indicate the model is either too lenient or too harsh on itself.

## Notes

- This evaluation uses the same model for both generation and judging, which is a form of self-evaluation
- The MT-Bench dataset is designed to test various aspects of model performance
- Results should be interpreted in the context of self-evaluation limitations
- Consider running multiple evaluations with different random seeds for more robust results
