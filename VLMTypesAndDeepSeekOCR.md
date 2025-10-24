## Dual-Tower Architecture (Vary)
```mermaid
graph TB
    IMG[Input Image] --> PRE[Dual Image Preprocessing]
    PRE --> |High-res| TOWER1[Vision Tower 1<br/>High Resolution]
    PRE --> |Low-res| TOWER2[Vision Tower 2<br/>Low Resolution]
    TOWER1 --> MERGE[Feature Merge]
    TOWER2 --> MERGE
    MERGE --> TOKENS[Vision Tokens]
    
    style PRE fill:#ffcccc
    style MERGE fill:#ccffcc
    
    PROS[✓ Controllable parameters<br/>✓ Multi-scale features]
    CONS[✗ Complex preprocessing<br/>✗ Dual image handling]
    
    style PROS fill:#d4edda
    style CONS fill:#f8d7da
```

## Tile-based (InternVL2.0): lower activation memory; many tokens / fragmentation
```mermaid
graph TB
    IMG[Input Image<br/>High Resolution] --> SPLIT[Split into Tiles]
    SPLIT --> T1[Tile 1]
    SPLIT --> T2[Tile 2]
    SPLIT --> T3[Tile 3]
    SPLIT --> TD[Tile ...]
    SPLIT --> TN[Tile N]
    
    T1 --> ENC[Vision Encoder]
    T2 --> ENC
    T3 --> ENC
    TD --> ENC
    TN --> ENC
    
    ENC --> TOKENS[Many Vision Tokens<br/>Fragmented Features]
    
    style SPLIT fill:#fff3cd
    style TOKENS fill:#ffcccc
    
    PROS[✓ Reduces activation memory<br/>✓ Handles high resolution]
    CONS[✗ Excessive fragmentation<br/>✗ Numerous vision tokens<br/>✗ Loss of global context]
    
    style PROS fill:#d4edda
    style CONS fill:#f8d7da
```

## Adaptive-resolution (Qwen2-VL): flexible resolution; high activation memory
```mermaid
graph TB
    IMG[Input Image<br/>Any Resolution] --> DETECT[Resolution Detection]
    DETECT --> ADAPT[Adaptive Encoder<br/>Dynamic Processing]
    
    ADAPT --> |Small| PATH1[Efficient Path]
    ADAPT --> |Medium| PATH2[Standard Path]
    ADAPT --> |Large| PATH3[High-Res Path]
    
    PATH1 --> TOKENS[Vision Tokens]
    PATH2 --> TOKENS
    PATH3 --> TOKENS
    
    TOKENS --> MEM[High Activation Memory]
    
    style ADAPT fill:#cce5ff
    style MEM fill:#ffcccc
    
    PROS[✓ Flexible resolution handling<br/>✓ Native resolution support<br/>✓ Better feature quality]
    CONS[✗ Massive activation memory<br/>✗ Higher computational cost]
    
    style PROS fill:#d4edda
    style CONS fill:#f8d7da
```

## Comparison Summary
```mermaid
graph LR
    subgraph Dual[Dual-Tower Vary]
        D1[2 Fixed Towers] -.- D2[Complex Preprocessing]
    end
    
    subgraph Tile[Tile-Based InternVL]
        T1[Split Image] -.- T2[Many Fragments]
    end
    
    subgraph Adaptive[Adaptive Qwen2-VL]
        A1[Dynamic Encoding] -.- A2[High Memory Use]
    end
    
    style Dual fill:#ffe6e6
    style Tile fill:#fff9e6
    style Adaptive fill:#e6f3ff
```

# DeepSeek-OCR Architecture Overview

## Key Concepts

**Core Innovation**: DeepSeek-OCR explores optical context compression by converting text into images, achieving 10-20× compression ratios. At 10× compression (1000 text tokens → 100 vision tokens), the model maintains 97% OCR accuracy, while 20× compression still retains 60% accuracy. This approach could enable efficient long-context processing by rendering older dialogue history into progressively smaller images, mimicking human memory decay.

**Architecture Design**: The 380M-parameter DeepEncoder combines an 80M SAM encoder (window attention for perception) with a 300M CLIP encoder (global attention for knowledge). A 16× convolutional compressor bridges them, reducing 4096 patches to 256 tokens before expensive global attention. This serial design maintains low activation memory at high resolutions. The decoder uses DeepSeek-3B-MoE with 570M active parameters.

**Practical Performance**: DeepSeek-OCR outperforms existing models with fewer tokens—beating GOT-OCR2.0 using just 100 tokens (vs 256) and MinerU2.0 using <800 tokens (vs 6000+). It supports multiple resolution modes (64-400+ tokens) in a single model and processes ~100 languages. Production deployment achieves 200K+ pages/day on a single A100-40G, with "deep parsing" capabilities for charts, formulas, geometry, and natural images.

## Overall DeepSeek-OCR Architecture
```mermaid
graph TB
    INPUT[Input Image<br/>Variable Resolution] --> DEEPENC[DeepEncoder 380M params]
    
    subgraph DeepEncoder Architecture
        DEEPENC --> SAM[SAM-base 80M<br/>Window Attention<br/>Perception Component]
        SAM --> PATCHES[4096 patches<br/>from 1024×1024 image]
        PATCHES --> CONV[16× Conv Compressor<br/>2-layer 3×3 kernel]
        CONV --> TOKENS256[256 vision tokens<br/>16× reduction]
        TOKENS256 --> CLIP[CLIP-large 300M<br/>Global Attention<br/>Knowledge Component]
    end
    
    CLIP --> VISTOK[Vision Tokens<br/>64-400+ tokens]
    VISTOK --> DECODER[DeepSeek-3B-MoE<br/>570M Active Params]
    PROMPT[Text Prompt] --> DECODER
    
    DECODER --> OUTPUT[Text Output<br/>OCR Result]
    
    style DEEPENC fill:#e1f5ff
    style SAM fill:#fff4e6
    style CLIP fill:#fff4e6
    style CONV fill:#ffe6e6
    style DECODER fill:#e6ffe6
```
## Vision Token Compression Strategy
```mermaid
graph TB
    subgraph Input Processing
        IMG[1024×1024 Image] --> PATCH[Patch Embedding<br/>16×16 patches]
        PATCH --> P4096[4096 tokens]
    end
    
    subgraph Window Attention Stage
        P4096 --> SAMPROC[SAM Window Attention<br/>Low activation memory<br/>80M params]
    end
    
    subgraph Compression Stage
        SAMPROC --> C1[Conv Layer 1<br/>stride=2, kernel=3<br/>256→512 channels]
        C1 --> C2[Conv Layer 2<br/>stride=2, kernel=3<br/>512→1024 channels]
        C2 --> COMP[256 tokens<br/>16× compression]
    end
    
    subgraph Global Attention Stage
        COMP --> CLIPPROC[CLIP Global Attention<br/>Dense processing<br/>300M params]
        CLIPPROC --> FINAL[256 vision tokens<br/>Ready for LLM]
    end
    
    style SAMPROC fill:#d4edda
    style COMP fill:#fff3cd
    style CLIPPROC fill:#cce5ff
```

## Compression-Decompression Concept
```mermaid
graph LR
    subgraph Text Domain
        TEXT[1000 text tokens<br/>Ground Truth]
    end
    
    subgraph Optical Compression
        TEXT --> RENDER[Render to Image]
        RENDER --> IMG[Document Image]
        IMG --> ENCODE[DeepEncoder]
        ENCODE --> COMP[100 vision tokens<br/>10× compression]
    end
    
    subgraph LLM Processing
        COMP --> LLM[DeepSeek-3B-MoE<br/>Decoder]
        PROMPT[OCR Prompt] --> LLM
    end
    
    subgraph Decompression
        LLM --> DECODE[Text Generation]
        DECODE --> RECON[~970 tokens<br/>97% accuracy]
    end
    
    TEXT -.->|10× compression| COMP
    COMP -.->|decode| RECON
    
    style COMP fill:#ffcccc
    style RECON fill:#ccffcc
    style TEXT fill:#cce5ff
```

## Multi-Resolution Support Modes
```mermaid
graph TB
    INPUT[Input Image] --> MODE{Resolution Mode}
    
    MODE --> TINY[Tiny Mode<br/>512×512<br/>64 tokens]
    MODE --> SMALL[Small Mode<br/>640×640<br/>100 tokens]
    MODE --> BASE[Base Mode<br/>1024×1024<br/>256 tokens]
    MODE --> LARGE[Large Mode<br/>1280×1280<br/>400 tokens]
    MODE --> GUNDAM[Gundam Mode<br/>Dynamic Tiling]
    MODE --> GUNDAM_M[Gundam-Master<br/>Ultra High-Res]
    
    subgraph Native Resolution
        TINY --> RESIZE1[Direct Resize]
        SMALL --> RESIZE2[Direct Resize]
        BASE --> PAD1[Padding to preserve<br/>aspect ratio]
        LARGE --> PAD2[Padding to preserve<br/>aspect ratio]
    end
    
    subgraph Dynamic Resolution
        GUNDAM --> TILE1[n×640 local tiles<br/>+ 1024 global view]
        GUNDAM_M --> TILE2[n×1024 local tiles<br/>+ 1280 global view]
        TILE1 --> ADAPTIVE1[n×100 + 256 tokens]
        TILE2 --> ADAPTIVE2[n×256 + 400 tokens]
    end
    
    RESIZE1 --> OUT[Vision Tokens]
    RESIZE2 --> OUT
    PAD1 --> OUT
    PAD2 --> OUT
    ADAPTIVE1 --> OUT
    ADAPTIVE2 --> OUT
    
    style GUNDAM fill:#ffe6e6
    style GUNDAM_M fill:#ffe6e6
```

## Training Pipeline
```mermaid
graph TB
    subgraph Stage 1: Train DeepEncoder
        DATA1[OCR 1.0 Data<br/>30M pages documents<br/>20M scene images] --> PRETRAIN[DeepEncoder Training]
        DATA2[OCR 2.0 Data<br/>10M charts<br/>5M formulas<br/>1M geometry] --> PRETRAIN
        DATA3[100M LAION<br/>general images] --> PRETRAIN
        
        PRETRAIN --> FREEZE[Freeze SAM + Compressor<br/>Train CLIP component]
    end
    
    subgraph Stage 2: Train Full Model
        FREEZE --> INIT[Initialize Full Model]
        
        OCR70[70% OCR Data] --> TRAIN[Joint Training]
        VISION20[20% General Vision] --> TRAIN
        TEXT10[10% Text-only Data] --> TRAIN
        
        INIT --> TRAIN
        
        TRAIN --> PP[Pipeline Parallel<br/>4 parts across GPUs]
    end
    
    subgraph Optional Stage 3
        TRAIN --> FINETUNE[Continue Training<br/>Gundam-Master mode<br/>6M samples]
    end
    
    PP --> MODEL[DeepSeek-OCR<br/>Final Model]
    FINETUNE --> MODEL_M[DeepSeek-OCR<br/>Gundam-Master]
    
    style PRETRAIN fill:#fff4e6
    style TRAIN fill:#e6ffe6
    style FINETUNE fill:#ffe6f0
```

## Deep Parsing Capability
```mermaid
graph TB
    INPUT[Document Image] --> FIRST[First Pass OCR<br/>Extract Structure]
    
    FIRST --> DETECT{Detect Elements}
    
    DETECT --> CHART[Chart Detected]
    DETECT --> FORMULA[Formula Detected]
    DETECT --> GEO[Geometry Detected]
    DETECT --> IMG[Natural Image]
    DETECT --> TEXT[Plain Text]
    
    CHART --> P2_CHART[Second Pass<br/>Parse chart to HTML]
    FORMULA --> P2_FORM[Second Pass<br/>Convert to SMILES]
    GEO --> P2_GEO[Second Pass<br/>Extract coordinates]
    IMG --> P2_IMG[Second Pass<br/>Dense caption]
    TEXT --> P2_TEXT[Direct OCR]
    
    P2_CHART --> OUTPUT[Structured Output<br/>Markdown + Parsed Elements]
    P2_FORM --> OUTPUT
    P2_GEO --> OUTPUT
    P2_IMG --> OUTPUT
    P2_TEXT --> OUTPUT
    
    style FIRST fill:#e1f5ff
    style P2_CHART fill:#ffe6e6
    style P2_FORM fill:#ffe6e6
    style P2_GEO fill:#ffe6e6
    style P2_IMG fill:#ffe6e6
```
