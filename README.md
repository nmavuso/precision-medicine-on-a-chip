**Low Latency Precision Medicine on a Chip**

We are devoloping an end to ed pipeline for streaming raw signals from an Oxford Nanopore Sequencer with a target of < 200 milliseconds latency. Stages include POD5 Files Reading with Z-core normalization, Long-Short-Term Memory (LSTM), a Recurrent Neural Network with 384 hidden Unites and 6 bidirection layers. Due to the limited resources in the Virtex FPGA, we use a layer tiling architecture for efficient use of limited BRAMs. 
What we are effectively doing is ping-ponging weights with a DDR3 (hot swapping and streaming them as they are needed for the specific stage of computation before they get swapped out. 

```Ocaml
  (**Stage 1:  POD5 BRAM Reader with Z-score normalization *)
  module Stage1_pod5_reader = struct
    open Hardcaml
    open Signal
    (** Configuration parameters *)
    module Config = struct
      type t =  {
          chunk_signal_samples : int;  (* 1050 samples *)
          data_width : int;   (*16 bits *)
          addr_width : int; (*log2(1050) = 11 bits*)
          mean_fixed : int; (*94.0 in fixed point *)
          stad_dev_fixed : int; (* 24.0 in finxed point *)
          fractional_bits : int;   (* Fixed point precision *)
      }

      let default =  {
          chunk_signal_sample = 1050;
          data_width = 16;
          addr_width = 11;
          mean_fixed = 94 lsl 8; (*Q8.8 formal : 94.0 * 256 *)
          std_dev_fixed = 24 lsl 8; (* Q8.8 format: 24.0 * 256 *)
          fractional_bits = 8;
      }
    end

   (** Input interface *)
    module I = struct
      type 'a t = {
            clock : 'a;
            clear : 'a;
          (* Control signals *)
          start : 'a ;
          start_read_idx : 'a; [@bits 10]
          num_reads : 'a; [@bits 10]

          (* Back pressure from downstream *)
          sample_ready : 'a;

          (* BRAM interface for reading POD5 data *)
          baram_dout : 'a; [@bits 16]
      } [@@deriving sexp_of, hardcaml]
    end

  (** Output interface *)
 module O = struct
    typer 'a t = {
        (* Read metadata *)
        read_id : 'a; [@bits 32]

        (* Normalized sample output *)
        sample_data : 'a; [@bits 16] (*Q8.8 fixed point normalized *)

        (* Control flags *)
        is_last_chunk : 'a;
        end_of_chunk : 'a;
        sample_valid : 'a;

        (*BRAM read address *)
        bram_addr : 'a; [@bits 11]
        bram_en : 'a;

        (* Status *)
        processing_done : 'a;
        reads_processed : 'a; [@bits 10]
        samples_processed : 'a; [@bits 32]
     }  [@@deriving sexp_of, hardcaml]
    end

    
  

    
  

```
