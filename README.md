
# Precision Medicine on a Chip: Real-Time FPGA Accelerated Sepsis Diagnosis

**Edge-Computing Point-of-Care Diagnostic System**

## Project Overview

This project implements a complete real-time sepsis diagnosis pipeline on FPGA hardware, achieving end-to-end latency of less than 200ms from nanopore sequencer to clinical result. The system performs mRNA basecalling, sequence alignment, gene expression quantification, pathogen identification, and ML-based sepsis classification entirely at the point of care. 

### Target Performance
- **End-to-End Latency:** < 200ms (MinION Sequencer → Diagnostic Result)
- **Target Devices:** 
  - Xilinx Zynq Ultrascale+ (16nm) with ARM Cortex-A53
  - Xilinx Virtex-7 (28nm)

---

## Architecture Overview

The pipeline consists of 11 stages spanning two FPGA devices:

1. **POD5 BRAM Reader** (Virtex-7) - Signal normalization
2. **Neural Network Basecaller** (Virtex-7) - Deep learning inference
3. **CRF Decoder** (MicroBlaze) - Beam search decoding
4. **FASTQ Formatter** (Zynq) - Sequence formatting
5. **QC Trimmer** (Zynq) - Quality-based read filtering
6. **Host & Pathogen Aligners** (Zynq) - Dual-path sequence alignment
7. **Gene Expression Quantifier** (Zynq) - Transcriptome analysis
8. **Pathogen Quantifier** (Zynq) - Microbial load tracking
9. **Biomarker Analyzer** (Zynq) - Diagnostic marker identification
10. **Sepsis ML Classifier** (Zynq) - Clinical decision support
11. **Report Generator** (ARM) - Human-readable output

---

## Stage 1: POD5 BRAM Reader with Normalization

### Overview
The first stage reads raw electrical signal data from nanopore sequencing and performs z-score normalization in real-time. This stage is critical for preparing data for the neural network basecaller.

### Hardware Platform
- **Chip:** Xilinx Virtex-7 (XC7VX485T)
- **Technology:** 28nm process

### Functionality

The POD5 reader processes mini-batches of raw signal samples and applies statistical normalization:

```verilog
pod5_bram_reader_norm #(
    .CHUNK_SIGNAL_SAMPLES(CHUNK_SIGNAL_SAMPLES)
) i_stage1_pod5_reader (
    .clk(clk),
    .reset_n(reset_n),
    .start(start_pipeline),
    .start_read_idx(10'd0),
    .num_reads(10'd1),
    .o_read_id(s1_read_id),
    .o_sample_data(s1_sample_data),
    .o_is_last_chunk(s1_is_last_chunk),
    .o_end_of_chunk(s1_end_of_chunk),
    .o_sample_valid(s1_sample_valid),
    .i_sample_ready(s1_sample_ready),
    .o_processing_done(),
    .o_reads_processed(),
    .o_samples_processed()
);
```

#### Key Operations

**Z-Score Normalization:**
```
normalized_value = (x - μ) / σ
```
- **Mean (μ):** 94.0
- **Standard Deviation (σ):** 24.0

This normalization standardizes the raw current measurements from the nanopore sensor, removing baseline drift and scaling variations between pores.

### Resource Utilization

| Resource | Usage | Percentage | Notes |
|----------|-------|------------|-------|
| **DSPs** | 4 | 0.1% | 2 for multiplication, 2 for addition |
| **BRAM** | 1 block | 0.1% | Signal buffer storage |
| **LUTs** | ~500 | 0.1% | Control logic and datapath |

### Memory Architecture

**Input Buffer Sizing:**
- Sample count: 1,050 samples per chunk
- Bit width: 16 bits per sample
- Total size: 16,800 bits
- BRAM requirement: 16,800 ÷ 36,864 = 0.45 → **1 block**

**Design Rationale:**  
The 1,050 sample chunk size is optimized for the downstream convolutional layers with stride-6 convolution, resulting in 175 timesteps (1050 ÷ 6 = 175) that align perfectly with the LSTM temporal processing.

### Data Flow

1. **Read Phase:** Fetch 1,050 consecutive samples from BRAM
2. **Normalization:** Apply z-score transformation using dedicated DSPs
3. **Streaming Output:** Emit normalized samples with valid/ready handshaking
4. **Chunk Management:** Signal end-of-chunk and last-chunk boundaries

### Key Parameters

```verilog
parameter CHUNK_SIGNAL_SAMPLES = 1050;
localparam INPUT_WIDTH = 16;      // 16-bit ADC samples
localparam OUTPUT_WIDTH = 16;     // 16-bit normalized float
```

### Performance Characteristics

- **Throughput:** 1,050 samples per chunk
- **Latency:** Negligible (<1ms) - pipelined operation
- **Precision:** 16-bit fixed-point arithmetic
- **Interface:** AXI-Stream compatible valid/ready protocol

---

## Stage 2: Neural Network Basecaller

### Overview
The neural network basecaller is the computational core of the pipeline, implementing a 6-layer deep learning model that converts normalized electrical signals into DNA base predictions. This stage represents the most resource-intensive component of the system.

### Hardware Platform
- **Chip:** Xilinx Virtex-7 (XC7VX485T)
- **Technology:** 28nm process

### Neural Network Architecture

The basecaller implements a hybrid CNN-LSTM architecture optimized for temporal sequence processing:

```verilog
fpga_basecaller_nn #(
    .CHUNK_SIGNAL_SAMPLES(CHUNK_SIGNAL_SAMPLES),
    .TIME_STEPS(NN_TIME_STEPS),
    .FEATURE_DIM(NN_FEATURE_DIM)
) i_stage2_basecaller_nn (
    .clk(clk),
    .reset_n(reset_n),
    .i_read_id(s1_read_id),
    .i_sample_data(s1_sample_data),
    .i_is_last_chunk(s1_is_last_chunk),
    .i_end_of_chunk(s1_end_of_chunk),
    .i_sample_valid(s1_sample_valid),
    .o_sample_ready(s1_sample_ready),
    .o_read_id(s2_read_id),
    .o_feature_data(s2_feature_data),
    .o_is_last_chunk(s2_is_last_chunk),
    .o_end_of_chunk(s2_end_of_chunk),
    .o_feature_valid(s2_feature_valid),
    .i_feature_ready(s2_feature_ready)
);
```

### Layer Configuration

#### Convolutional Layers

**Layer 1: Initial Feature Extraction**
- Input channels: 1
- Output channels: 16
- Kernel size: 5
- Stride: 1
- Activation: Swish (x · sigmoid(x))
- Purpose: Extract local signal patterns

**Layer 2: Feature Refinement**
- Input channels: 16
- Output channels: 16
- Kernel size: 5
- Stride: 1
- Activation: Swish
- Purpose: Refine and denoise features

**Layer 3: Temporal Downsampling**
- Input channels: 16
- Output channels: 384
- Kernel size: 19
- Stride: 6 (critical for temporal reduction)
- Activation: Tanh
- Purpose: Compress temporal dimension while expanding feature space

#### LSTM Layers

**Bidirectional LSTM Stack:**
- Number of layers: 6
- Hidden units per layer: 384
- Direction: Bidirectional (forward + backward processing)
- Purpose: Capture long-range temporal dependencies in both directions

The LSTM processes sequences in both forward and reverse directions, allowing the network to leverage both past and future context when predicting each base.

### Resource Utilization

| Resource | Usage | Percentage | Critical Notes |
|----------|-------|------------|----------------|
| **DSPs** | 2,500 | 89.3% | **Near maximum - critical constraint** |
| **BRAM** | 977 blocks | 94.8% | **Near maximum - critical constraint** |
| **LUTs** | ~175,000 | 36% | Sufficient headroom |

**Design Challenge:** Both DSP and BRAM utilization exceed 89%, requiring careful optimization and leaving minimal room for expansion.

### Memory Architecture and Layer Tiling

#### Weight Storage Strategy

The network employs a **layer-tiling architecture** due to BRAM constraints:

- **Total weight size:** 4.5 MB (36 Mb)
- **On-chip BRAM capacity:** 977 blocks × 36,864 bits = 36 Mb
- **Strategy:** Load one layer at a time from external DDR3 memory

**BRAM Calculation:**
```
Total weights: 4.5 MB = 36,000,000 bits
BRAM requirement: 36,000,000 ÷ 36,864 = 976.5 → 977 blocks
```

This consumes nearly all available BRAM, requiring precise memory management.

#### DDR3 Memory Interface

**External Memory Requirements:**
- **Bandwidth:** 300 MB/s (81.7% of theoretical 367 MB/s)
- **Latency:** 15.0 ms per layer load
- **Interface:** DDR3-1866 through Xilinx MIG (Memory Interface Generator)

**Per-Layer Timing:**
- Weight loading from DDR3: 15.0 ms
- Computation: 0.13 ms (dominated by I/O)
- Total per layer: 15.13 ms
- **Total for 6 layers: 90.8 ms**

### LUT Breakdown

The 175,000 LUT utilization breaks down as follows:

| Component | LUT Usage | Function |
|-----------|-----------|----------|
| DDR3 MIG Controller | ~60,000 | Memory interface logic |
| AXI Interconnect | ~20,000 | Data movement fabric |
| Non-linear Activations | ~30,000 | Swish and Tanh functions |
| Control Logic | ~50,000 | FSMs and sequencing |
| Systolic Array | ~15,000 | Matrix multiplication engine |

### Computational Architecture

#### Systolic Array Design

The core computation uses a systolic array for efficient matrix multiplication:
- Parallel multiply-accumulate operations
- Data reuse to minimize memory bandwidth
- Pipelined for maximum throughput

#### Activation Functions

**Swish Activation (Layers 1-2):**
```
swish(x) = x · sigmoid(x)
```
Implemented using LUT-based approximation with 30,000 LUTs for high-throughput parallel computation.

**Tanh Activation (Layer 3):**
```
tanh(x) = (e^x - e^-x) / (e^x + e^-x)
```
Piecewise linear approximation for FPGA efficiency.

### Input/Output Specifications

#### Input
- **Shape:** [1050 samples]
- **Format:** 16-bit normalized floats from Stage 1
- **Rate:** Streaming with backpressure

#### Output
- **Shape:** [175 timesteps × 384 features]
- **Total elements:** 67,200 float32 values
- **Size:** 268.8 KB per mini-batch
- **Format:** Streaming feature vectors for decoder

**Temporal Reduction:**
```
1050 samples → (stride 6 convolution) → 175 timesteps
```

This 6× reduction is crucial for making the subsequent LSTM processing tractable while preserving essential temporal information.

### Key Parameters

```verilog
parameter CHUNK_SIGNAL_SAMPLES = 1050;
parameter NN_TIME_STEPS = 175;        // 1050 ÷ 6
parameter NN_FEATURE_DIM = 384;
```

### Performance Characteristics

- **Throughput:** One 1050-sample chunk per 90.8 ms
- **Bottleneck:** DDR3 weight loading (15 ms × 6 layers)
- **Compute efficiency:** Only 0.85% of time spent in computation
- **Optimization opportunity:** Compute is fast; I/O dominates

### Critical Design Trade-offs

1. **Layer Tiling vs. Performance:** External memory access adds 90ms latency but enables a larger model than on-chip BRAM would allow

2. **DSP Allocation:** 89.3% utilization maximizes parallelism while leaving minimal margin for error or expansion

3. **Bidirectional LSTMs:** Double the computation cost but significantly improve basecalling accuracy by using future context

4. **Feature Dimension (384):** Balances model capacity with resource constraints—larger would improve accuracy but exceed BRAM limits

---

## Stages 3-11: Pipeline Overview

The remaining stages complete the diagnostic pipeline:

### Stage 3: CRF Decoder (MicroBlaze Softcore)
Beam search decoding with CRF layer converts LSTM features to ACTG bases with quality scores.

### Stage 4-5: FASTQ Processing (Zynq ZCU102)
Formatting and quality-based trimming of sequencing reads.

### Stage 6: Dual-Path Alignment (Zynq ZCU102)
Simultaneous alignment to host transcriptome (20K genes) and pathogen genomes (100 species).

### Stage 7-8: Quantification (Zynq ZCU102)
Gene expression levels (TPM) and pathogen abundance calculation.

### Stage 9-10: Clinical Decision Support (Zynq ZCU102)
Biomarker analysis and ML-based sepsis classification with confidence scoring.

### Stage 11: Report Generation (ARM Cortex-A53)
Clinical report formatting in software on Zynq Processing System.

---

## System Integration

### Two-Chip Architecture

**Virtex-7 (Stages 1-3):**
- Focus: Computationally intensive basecalling
- Nearly maximal resource utilization
- Dedicated to signal processing and neural inference

**Zynq Ultrascale+ (Stages 4-11):**
- Focus: Bioinformatics and clinical analysis
- Hardware-software co-design
- ARM processing for flexible report generation


