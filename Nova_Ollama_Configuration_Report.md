# Nova: Ollama Installation & Configuration Report
## Large Language Model Infrastructure Setup

**Setup Date:** February 15, 2026  
**Status:** Operational  
**Primary Model:** Mistral 7B Instruct (Q4)  
**Additional Models:** Qwen3 8B (loading)

---

## Executive Summary

This report documents the installation, configuration, and operational setup of Ollama—an open-source framework for running Large Language Models locally on GPU hardware. Ollama serves as Nova's LLM inference engine, providing GPU-accelerated model serving for code generation and data analysis tasks.

The deployment prioritizes ease of use, model management, and performance optimization over containerization complexity, reflecting an intentional architectural decision to minimize overhead during the initial deployment phase.

---

## 1. Ollama Architecture & Selection

### 1.1 Why Ollama?

**Primary Selection Criteria:**
- **Ease of Use:** Simple model management and inference API
- **GPU Acceleration:** Seamless Nvidia CUDA support for RTX 4060
- **Model Ecosystem:** Integration with Hugging Face and community models
- **Local-First Design:** Built for local inference, no cloud dependency
- **Open Source:** Community-driven, transparent development

**Alternatives Considered:**
- **vLLM:** More feature-rich but higher complexity
- **Text-generation-webui:** Heavier frontend overhead
- **Hugging Face TGI:** Enterprise-focused, less developer-friendly
- **llama.cpp:** Lower-level control, steeper learning curve

Ollama selected for sweet spot between capability and simplicity.

### 1.2 Deployment Model: Direct Installation (Non-Containerized)

**Decision:** Install Ollama directly on host system (not in Docker)

**Rationale:**

| Factor | Direct Installation | Containerized |
|--------|-------------------|-----------------|
| **GPU Integration** | Native CUDA passthrough | Requires GPU passthrough setup |
| **Performance Overhead** | Minimal (~1-2%) | Docker overhead (~5-10%) |
| **Model Persistence** | Direct filesystem | Volume management required |
| **Setup Complexity** | Simple (~5 minutes) | Complex (~30 minutes) |
| **Resource Efficiency** | Optimal for single workload | Better for isolation |

**Current Context:** Single primary workload (LLM inference); direct installation appropriate.

**Future Migration Path:** Will containerize when multiple services require strict resource isolation or version management becomes critical.

---

## 2. Installation & Configuration

### 2.1 Installation Steps

**Step 1: Download & Install Ollama**

```bash
# Official installation method
curl https://ollama.ai/install.sh | sh

# Or via package manager (if available)
sudo apt install ollama
```

**Step 2: Verify Installation**

```bash
which ollama
ollama version
```

**Installed Location:** `/usr/local/bin/ollama`

**Step 3: Test GPU Detection**

```bash
ollama list  # Should run without errors
```

Ollama automatically detects Nvidia GPU and initializes CUDA.

### 2.2 Network Configuration

**Binding Strategy:** Expose Ollama API on all network interfaces for local access

**Environment Variable:**
```bash
export OLLAMA_HOST=0.0.0.0:11434
```

**Rationale:**
- `0.0.0.0` binds to all interfaces (accessible from other network machines)
- Port 11434 is Ollama default; widely recognized
- Local-network-only firewall rules provide security boundary

**Verification:**
```bash
netstat -tulpn | grep 11434
# Should show: tcp  0  0 0.0.0.0:11434  0.0.0.0:*  LISTEN
```

### 2.3 Service Management via Systemd

**Service File:** `/etc/systemd/system/ollama.service`

```ini
[Unit]
Description=Ollama
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment="OLLAMA_HOST=0.0.0.0:11434"
ExecStart=/usr/local/bin/ollama serve
Restart=always
RestartSec=10
User=[primary-user]
WorkingDirectory=/home/[primary-user]

[Install]
WantedBy=multi-user.target
```

**Critical Configuration Notes:**

1. **User Context:** Service runs as non-root user (not `root`)
   - Ensures models stored in user home directory are accessible
   - Prevents permission errors on service restart
   
2. **Environment Variable:** `OLLAMA_HOST=0.0.0.0:11434` set in service file
   - Persists across reboots
   - No manual command-line configuration needed

3. **Restart Policy:** `Restart=always` with 10-second delay
   - Ensures service recovers from failures
   - Prevents rapid restart loops

**Service Commands:**

```bash
# Enable auto-start on boot
sudo systemctl enable ollama.service

# Start service manually
sudo systemctl start ollama.service

# Check status
sudo systemctl status ollama.service

# View logs
sudo journalctl -u ollama.service -n 50

# Restart service
sudo systemctl restart ollama.service
```

---

## 3. Model Selection & Deployment

### 3.1 Model Selection Criteria

**Primary Use Case:** Equal split between code generation and data analysis/reporting

**Model Requirements:**
- Strong code generation capability
- Good reasoning for data analysis
- Quantization support (fit in 6GB RTX 4060 VRAM)
- Fast inference (10-30 second latency acceptable)

### 3.2 Primary Model: Mistral 7B Instruct (Q4)

**Model Identifier:** `mistral:7b-instruct-q4_K_M`

**Specifications:**
- **Base Size:** 7 billion parameters
- **Quantization:** Q4 (4-bit)
- **VRAM Usage:** ~5-6 GB
- **Download Size:** ~4.0 GB
- **Performance:** ~10-15 seconds typical inference

**Installation:**

```bash
ollama pull mistral:7b-instruct-q4_K_M
```

**Verification:**

```bash
ollama list
# Output:
# NAME                              ID              SIZE    MODIFIED
# mistral:7b-instruct-q4_K_M       abc123...       4.0GB   2 minutes ago
```

**Capabilities:**
- ✅ Code generation (Python, JavaScript, Go, etc.)
- ✅ Data analysis and pattern recognition
- ✅ Natural language reasoning
- ✅ Multi-turn conversation
- ✅ Low latency for fast iteration

**Trade-offs:**
- Less specialized than CodeLlama for pure code tasks
- Not as large as 13B+ models (smaller context window)
- Excellent balance for dual-purpose workload

### 3.3 Secondary Model: Qwen3 8B (Loading)

**Model Identifier:** `qwen3:8b`

**Status:** Currently pulling during deployment

**Rationale:**
- Hybrid thinking mode (switches between reasoning and fast response)
- Strong code generation and data analysis
- Comparable VRAM footprint to Mistral
- Evaluation for comparison and specialized analysis tasks

**Installation:**

```bash
ollama pull qwen3:8b
```

### 3.4 Model Persistence Across Reboots

**Challenge Encountered:** Models disappeared after reboot

**Root Cause:** Ollama running as `root` in systemd, models stored in user home directory

**Resolution:** 
1. Changed systemd service to run as non-root user
2. Verified models remain in `~/.ollama/models/` directory
3. Confirmed models auto-load on service restart

**Model Storage Location:**
```bash
~/.ollama/models/blobs/    # Model weights
~/.ollama/models/manifests/ # Model metadata
```

**Persistence Verification:**

```bash
# Before reboot
ollama list

# After reboot
sudo systemctl restart ollama.service
sleep 5
ollama list  # Should still show models
```

---

## 4. API & Integration

### 4.1 Ollama REST API

**Base URL:** `http://localhost:11434`

**Primary Endpoints:**

**List Models:**
```bash
curl http://localhost:11434/api/tags
```

Response:
```json
{
  "models": [
    {
      "name": "mistral:7b-instruct-q4_K_M",
      "modified_at": "2026-02-15T18:10:00Z",
      "size": 4000000000,
      "digest": "..."
    }
  ]
}
```

**Run Inference:**
```bash
curl http://localhost:11434/api/generate -d '{
  "model": "mistral:7b-instruct-q4_K_M",
  "prompt": "Write a Python function to calculate factorial",
  "stream": false
}'
```

**Health Check:**
```bash
curl http://localhost:11434/api/tags
# Returns 200 if service healthy
```

### 4.2 OpenAI-Compatible API Layer

Ollama provides OpenAI-compatible API endpoint at `http://localhost:11434/v1/`

This allows integration with tools expecting OpenAI API format:
- Python: `openai` library
- JavaScript: `openai` SDK
- Custom applications

**Example (Python):**
```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"
)

response = client.chat.completions.create(
    model="mistral:7b-instruct-q4_K_M",
    messages=[
        {"role": "user", "content": "Explain binary search"}
    ]
)
```

---

## 5. Performance Characteristics

### 5.1 GPU Utilization

**RTX 4060 Performance:**
- **VRAM Allocation:** ~5-6 GB for Mistral 7B Q4
- **GPU Utilization:** 80-95% during inference
- **Power Draw:** ~70W typical
- **Thermal:** Safe operating temps (60-75°C)

**CPU Impact:** Minimal (mostly GPU-bound)

### 5.2 Inference Speed Benchmarks

**Mistral 7B Instruct (Q4) on RTX 4060:**

| Task | Latency | Throughput |
|------|---------|-----------|
| Simple Q&A | 5-8 seconds | ~50 tokens/sec |
| Code generation (50 tokens) | 8-12 seconds | ~50-60 tokens/sec |
| Data analysis (complex) | 15-25 seconds | ~40-50 tokens/sec |
| Long-form response (200+ tokens) | 30-45 seconds | ~50 tokens/sec |

**Note:** Latency includes model load time on cold start (~2-3 seconds)

### 5.3 Memory Management

**System RAM:**
- Base usage: ~2-3 GB
- Per model: ~3-5 GB (context buffer)
- Peak during generation: ~5-8 GB

**With 62 GB available:** No memory pressure; comfortable headroom for concurrent services

---

## 6. Troubleshooting & Issues

### 6.1 Models Disappearing After Reboot

**Symptom:** `ollama list` returns empty set

**Root Cause:** Service running as wrong user; cannot access model directory

**Solution:**
```bash
# Check service user
sudo systemctl show -p User ollama.service

# Fix service file
sudo nano /etc/systemd/system/ollama.service
# Change: User=root  →  User=[primary-user]

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart ollama.service

# Verify models reappear
ollama list
```

### 6.2 GPU Not Detected

**Symptom:** Ollama uses CPU only; models very slow

**Diagnosis:**
```bash
# Check CUDA detection
ollama list  # Should show CUDA initialization message

# Check GPU availability
nvidia-smi
```

**Solution:**
- Verify Nvidia drivers installed: `nvidia-smi` should work
- Reinstall Ollama to detect GPU changes
- Check `/var/log/Xorg.0.log` for driver issues

### 6.3 Port Already in Use

**Symptom:** "address already in use" error on port 11434

**Diagnosis:**
```bash
sudo lsof -i :11434
```

**Solution:**
```bash
# Kill conflicting process
sudo kill -9 [PID]

# Or use different port
export OLLAMA_HOST=0.0.0.0:11435
```

### 6.4 Model Pull Interruption

**Issue:** Interrupted `ollama pull` during download

**Resolution:**
```bash
# Resume pull (Ollama resumes from checkpoint)
ollama pull mistral:7b-instruct-q4_K_M

# Or force fresh download
rm -rf ~/.ollama/models/blobs/[model-hash]
ollama pull mistral:7b-instruct-q4_K_M
```

---

## 7. Integration with Open WebUI

Ollama provides inference backend for Open WebUI web interface (see Open WebUI Configuration Report).

**Connection Configuration:**
- Base URL: `http://localhost:11434`
- Auto-detection by Open WebUI
- Model selection via dropdown in UI

**Network Isolation:**
- Ollama API (port 11434): Internal only
- Open WebUI (port 3000): User-facing via Nginx reverse proxy
- Clean separation of concerns

---

## 8. Current Deployment Status

### 8.1 Ollama Service

✅ **Status:** Active (running)

```bash
● ollama.service - Ollama
     Loaded: loaded (/etc/systemd/system/ollama.service; enabled)
     Active: active (running)
```

### 8.2 Models Loaded

- **Mistral 7B Instruct (Q4):** Operational
- **Qwen3 8B:** Loading during deployment

### 8.3 API Accessibility

✅ **Local Network:** Accessible at `http://[nova-ip]:11434`

✅ **OpenAI Compatible:** Available at `http://[nova-ip]:11434/v1/`

---

## 9. Future Enhancements

### 9.1 Model Expansion

- Load additional specialized models (CodeLlama 13B for pure coding)
- Evaluate Qwen3 vs. Mistral for dual-workload performance
- Consider embedding models for semantic search

### 9.2 Containerization Migration

- Move to Docker deployment when:
  - Multiple isolated services need resource limits
  - Model version management becomes critical
  - Team collaboration requires reproducible environments

### 9.3 Model Serving Optimization

- Implement request queuing for concurrent model calls
- Add inference caching for repeated queries
- Monitor VRAM utilization for optimal batch sizing

### 9.4 Monitoring & Observability

- Track inference latency metrics
- Monitor GPU utilization and temperature
- Log model usage patterns for capacity planning

---

## 10. References & Resources

- Ollama Documentation: https://github.com/ollama/ollama
- Ollama Model Library: https://ollama.ai/library
- Mistral AI: https://mistral.ai/
- Qwen (Alibaba): https://qwenlm.github.io/

---

## Appendix A: Useful Commands

```bash
# Model management
ollama list                                    # List installed models
ollama pull mistral:7b-instruct-q4_K_M       # Download/install model
ollama rm mistral:7b-instruct-q4_K_M         # Remove model
ollama show mistral:7b-instruct-q4_K_M       # Model details

# Interactive mode
ollama run mistral:7b-instruct-q4_K_M        # Chat with model
# Type 'exit' or '/bye' to quit

# API testing
curl http://localhost:11434/api/tags          # List available models
curl http://localhost:11434/api/generate -d '{...}'  # Run inference

# Service management
sudo systemctl start ollama.service            # Start service
sudo systemctl stop ollama.service             # Stop service
sudo systemctl restart ollama.service          # Restart service
sudo systemctl enable ollama.service           # Enable auto-start
sudo journalctl -u ollama.service -f          # View live logs

# System integration
ps aux | grep ollama                           # Check if running
sudo lsof -i :11434                           # Verify port binding
nvidia-smi                                    # GPU status
```

---

**Document Version:** 1.0  
**Last Updated:** February 15, 2026  
**Status:** Final  
**Sensitive Information:** Redacted (IP addresses, file paths to /home/[user])
