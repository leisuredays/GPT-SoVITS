# GPT-SoVITS Performance Documentation

This directory contains comprehensive performance analysis, optimization guides, and configuration documentation for the GPT-SoVITS TTS system.

## 📚 Documentation Index

### Performance Analysis
- **[gpu_vs_cpu_comparison.md](gpu_vs_cpu_comparison.md)** - Complete GPU vs CPU performance comparison with detailed metrics, cost analysis, and recommendations
- **[model_speed_comparison_result.md](model_speed_comparison_result.md)** - Custom model vs v2 base model speed test results
- **[model_size_comparison.md](model_size_comparison.md)** - Analysis of different model versions and their performance characteristics

### Optimization Guides
- **[gpt_optimization_guide.md](gpt_optimization_guide.md)** - GPT model optimization strategies including sampling parameters (top_k, temperature, repetition_penalty)
- **[cpu_mode_guide.md](cpu_mode_guide.md)** - Complete guide for running TTS on CPU with performance expectations and optimization tips
- **[streaming_server_control.md](streaming_server_control.md)** - Server-side streaming mode configuration and control methods

### Configuration Reference
- **[batch_size_explained.md](batch_size_explained.md)** - Detailed explanation of batch_size parameter and its impact on streaming performance
- **[api실행.md](api실행.md)** - API execution basic guide (Korean)

## 🎯 Quick Reference

### Performance Summary

| Mode | Device | TTFB | Total Time | Use Case |
|------|--------|------|------------|----------|
| **Production** | GPU (CUDA + FP16) | 0.422s | 1.369s | Real-time chat, voice assistants |
| **Development** | CPU (FP32) | 2.080s | 4.270s | Testing, low-frequency requests |
| **Custom Model** | GPU | 0.422s | 1.369s | Best quality/speed balance |
| **v2 Base** | GPU | 0.367s | 1.201s | Slightly faster, less quality |

### Key Optimization Settings

#### Fastest Response (Production)
```yaml
# tts_infer.yaml
device: cuda
is_half: true
version: v2ProPlus
```

```python
# API Request
{
    "streaming_mode": 3,
    "batch_size": 1,
    "top_k": 8,
    "temperature": 0.85
}
```

#### Best Quality
```python
{
    "streaming_mode": 1,
    "batch_size": 1,
    "top_k": 15,
    "temperature": 1.0
}
```

## 📊 Test Results Archive

All test scripts, audio files, and raw results have been moved to `.archive/`:
- `.archive/test_scripts/` - Python test scripts
- `.archive/test_audio/` - Generated test audio files (50MB+)
- `.archive/test_results/` - JSON results and chunk files
- `.archive/logs/` - Server log files

## 🔍 Key Findings

1. **GPU vs CPU**: GPU is 4.9x faster for TTFB (0.422s vs 2.080s)
2. **Model Size Impact**: Custom 165MB model only 15% slower than 102MB v2 base
3. **Bottleneck**: GPT model (149MB) accounts for 40% of inference time
4. **Optimization**: Sampling parameters can provide 20-40% speedup
5. **Cost Efficiency**: GPU is 3.2x more cost-efficient per request in cloud

## 🚀 Recommended Setup

**For Real-time Applications:**
- GPU mode with custom fine-tuned model
- streaming_mode: 3
- batch_size: 1
- TTFB target: < 0.5s ✅

**For Batch Processing:**
- CPU or GPU mode depending on availability
- streaming_mode: 0 (disabled)
- batch_size: 4-8 for higher throughput

## 📝 Related Files

- Configuration: `/GPT_SoVITS/configs/tts_infer.yaml`
- API Server: `/api_v2.py`
- Reference Audio: `/yuzuha_reference.wav`

---

**Last Updated**: 2026-01-18
**Test Environment**: WSL2, Ubuntu, RTX GPU, GPT-SoVITS v2ProPlus
