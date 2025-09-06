# OpenShift AI Evals and Guardrails

This repository contains a complete setup for deploying AI models with guardrails and content filtering capabilities on OpenShift AI (RHOAI). The demo showcases how to integrate TrustyAI guardrails with LLM inference services for safe AI deployment.

## **Overview**

This demo deploys:
- **Llama 3.2 3B Instruct** - A large language model for text generation
- **Granite Guardian HAP 125M** - A harmful activity prevention model for content filtering
- **TrustyAI Guardrails Orchestrator** - Content filtering and safety orchestration
- **FMS Orchestr8** - Model serving and routing configuration

## **Prerequisites**

### **Required Infrastructure**
- **OpenShift AI (RHOAI) 2.22.1** deployed and running
- **OpenShift 4.19** cluster with GPU support
- **NVIDIA GPUs** available for model inference
- **OC CLI** installed and configured
- **Cluster admin privileges** for resource creation

## Check out this repo to follow instruction on how to setup Red Hat OpenShift AI
### *https://github.com/ravisharma5/OpenShiftAISetup*

### **System Requirements**
- **GPU-enabled nodes** with NVIDIA drivers
- **Sufficient resources** for model serving (see resource requirements below)
- **Storage** for model artifacts and data persistence

## **Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                    Guardrails Gateway                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │  Regex Detector │  │   HAP Detector  │  │  Passthrough│ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                NLP Orchestrator                            │
│  ┌─────────────────────────────────────────────────────────┐│
│  │         Chat Generation Service                         ││
│  │     (Llama 3.2 3B Instruct)                            ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## **Quick Start**

### **1. Clone and Navigate**
```bash
git clone <your-repo-url>
cd connect-evals-guardrails-demo
```

### **2. Create Models Namespace**
```bash
oc create namespace models
```

### **3. Deploy Serving Runtimes**
```bash
# Deploy vLLM runtime for Llama model
oc apply -f deploy-models/llama-32-3b-instruct-servingruntime.yaml

# Deploy HuggingFace runtime for HAP model
oc apply -f deploy-models/granite-guardian-hap-125m-servingruntime.yaml
```

### **4. Deploy Inference Services**
```bash
# Deploy Llama 3.2 3B Instruct model
oc apply -f deploy-models/llama-32-3b-instruct-inferenceservice.yaml

# Deploy Granite Guardian HAP model
oc apply -f deploy-models/granite-guardian-hap-125m-inferenceservice.yaml
```

### **5. Deploy Guardrails Configuration**
```bash
# Deploy configuration files
oc apply -f guardrails-setup/fms-orchestr8-config-gateway.yaml
oc apply -f guardrails-setup/fms-orchestr8-config-nlp.yaml
oc apply -f guardrails-setup/guardrails-orchestrator-config.yaml

# Deploy guardrails orchestrator
oc apply -f guardrails-setup/guardrails-orchestrator.yaml
```

## **Resource Requirements**
#### Below resources are for example. If you are using quantised models you can save on spending GPUs and still get good performance.

### **Llama 3.2 3B Instruct**
- **CPU**: 1 core (request/limit)
- **Memory**: 4Gi (request/limit)
- **GPU**: 2 NVIDIA GPUs
- **Storage**: ~6GB for model artifacts

### **Granite Guardian HAP 125M**
- **CPU**: 4 cores (request), 8 cores (limit)
- **Memory**: 8Gi (request), 10Gi (limit)
- **Storage**: ~500MB for model artifacts

## **Configuration Details**

### **Guardrails Configuration**

The guardrails system includes:

1. **Regex Detector**: Filters content based on predefined patterns
   - Example: Blocks fruit-related content (`orange`, `apple`, `cranberry`, etc.)

2. **HAP Detector**: Uses the Granite Guardian model for harmful activity prevention
   - Detects harmful, abusive, or problematic content
   - Threshold: 0.5 (configurable)

3. **Routing Rules**:
   - **All route**: Applies both detectors
   - **HAP route**: Only applies HAP detector
   - **Passthrough route**: No filtering

### **Model Configuration**

#### **Llama 3.2 3B Instruct**
- **Framework**: vLLM with CUDA acceleration
- **Precision**: Half-precision (FP16)
- **Max Length**: 20,000 tokens
- **GPU Memory**: 95% utilization
- **Features**: Chunked prefill, auto tool choice, JSON tool calls

#### **Granite Guardian HAP 125M**
- **Framework**: HuggingFace Transformers
- **Purpose**: Content safety and harmful activity prevention
- **Input/Output**: Text content analysis
- **Threshold**: 0.5 (configurable)

## **Verification**

### **Check Model Deployment**
```bash
# Check inference services
oc get inferenceservice -n models

# Check serving runtimes
oc get servingruntime -n models

# Check pods
oc get pods -n models
```

### **Check Guardrails Orchestrator**
```bash
# Check guardrails orchestrator
oc get guardrailsorchestrator -n models

# Check orchestrator pods
oc get pods -l app=guardrails-orchestrator -n models
```

### **Test Model Endpoints**
```bash
# Get Llama model URL
oc get route llama-32-3b-instruct -n models

# Get HAP model URL
oc get route granite-guardian-hap-125m -n models
```

## **Usage Examples**

### **Direct Model Access**
```bash
# Test Llama model directly
curl -X POST "https://llama-32-3b-instruct-models.apps.your-cluster.com/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-32-3b-instruct",
    "messages": [{"role": "user", "content": "Hello, how are you?"}]
  }'

# Test HAP model directly
curl -X POST "https://granite-guardian-hap-125m-models.apps.your-cluster.com/v1/predict" \
  -H "Content-Type: application/json" \
  -d '{"inputs": "This is a test message"}'
```

### **Through Guardrails Gateway**
```bash
# Test with guardrails filtering
curl -X POST "https://guardrails-gateway-models.apps.your-cluster.com/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-32-3b-instruct",
    "messages": [{"role": "user", "content": "Tell me about apples"}]
  }'
```

## **Customization**

### **Modify Detector Rules**
Edit `guardrails-setup/fms-orchestr8-config-gateway.yaml`:
```yaml
detectors:
  - name: custom_regex
    input: true
    output: true
    detector_params:
      regex:
        - \b(?i:your|custom|pattern)\b
```

### **Adjust Detection Thresholds**
Edit `guardrails-setup/fms-orchestr8-config-nlp.yaml`:
```yaml
detectors:
  hap:
    default_threshold: 0.7  # Increase for stricter filtering
```

### **Scale Resources**
Edit the inference service files to adjust:
- CPU and memory limits
- GPU requirements
- Replica counts
- Model parameters

## **Troubleshooting**

### **Common Issues**

1. **Models not starting**:
   - Check GPU availability: `oc describe nodes | grep nvidia.com/gpu`
   - Verify resource limits and requests
   - Check pod logs: `oc logs -f deployment/llama-32-3b-instruct-predictor -n models`

2. **Guardrails not working**:
   - Verify ConfigMaps are applied: `oc get configmap -n models`
   - Check orchestrator logs: `oc logs -f deployment/guardrails-orchestrator -n models`
   - Ensure service endpoints are correct

3. **GPU not detected**:
   - Verify NFD instance is available: `oc get nfd-instance -n openshift-nfd`
   - Check GPU operator status: `oc get clusterpolicy`
   - Ensure nodes have GPU labels

### **Useful Commands**
```bash
# Check all resources
oc get all -n models

# View detailed status
oc describe inferenceservice llama-32-3b-instruct -n models

# Check events
oc get events -n models --sort-by='.lastTimestamp'

# Monitor logs
oc logs -f deployment/llama-32-3b-instruct-predictor -n models
```

## **Security Considerations**

- **Authentication**: Models are secured with OpenShift authentication
- **Network Policies**: Consider implementing network policies for additional security
- **Resource Limits**: Set appropriate CPU/memory limits to prevent resource exhaustion
- **Image Security**: Use trusted container images and scan for vulnerabilities

## **Monitoring and Observability**

- **Prometheus Metrics**: Available on port 8080 for both models
- **OpenTelemetry**: Configured for distributed tracing
- **Logs**: Available through OpenShift logging stack
- **Dashboards**: Access through OpenShift AI dashboard

## **Next Steps**

1. **Customize Detection Rules**: Modify regex patterns and thresholds
2. **Add More Models**: Deploy additional models with guardrails
3. **Implement Monitoring**: Set up alerts and dashboards
4. **Scale Deployment**: Adjust replicas based on load
5. **Security Hardening**: Implement additional security measures
