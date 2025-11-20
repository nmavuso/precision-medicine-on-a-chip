**Low Latency Precision Medicine on a Chip**

We are devoloping an end to ed pipeline for streaming raw signals from an Oxford Nanopore Sequencer with a target of < 200 milliseconds latency. Stages include POD5 Files Reading with Z-core normalization, Long-Short-Term Memory (LSTM), a Recurrent Neural Network with 384 hidden Unites and 6 bidirection layers. Due to the limited resources in the Virtex FPGA, we use a layer tiling architecture for efficient use of limited BRAMs. 
What we are effectively doing is ping-ponging weights with a DDR3 (hot swapping and streaming them as they are needed for the specific stage of computation before they get swapped out. 

```SystemVerilog
  (**Stage 1:  POD5 BRAM Reader with Z-score normalization *)
  module Stage1_pod5_reader = struct
    open Hardcaml


```
