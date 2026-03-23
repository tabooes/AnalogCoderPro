# AnalogCoder-Pro: Unifying Analog Circuit Generation and Optimization via Multi-modal LLMs


[Yao Lai](https://laiyao1.github.io/)<sup>1</sup>, [Souradip Poddar](https://www.linkedin.com/in/souradip-poddar-52376212a/)<sup>2</sup>, [Sungyoung Lee](https://brianlsy98.github.io/)<sup>2</sup>, [Guojin Chen](https://gjchen.me/)<sup>3</sup>, [Mengkang Hu](https://aaron617.github.io/)<sup>1</sup>, [Bei Yu](https://www.cse.cuhk.edu.hk/~byu/)<sup>3</sup>, [Ping Luo](http://luoping.me/)<sup>1</sup>, [David Z. Pan](https://users.ece.utexas.edu/~dpan/)<sup>2</sup>.

<sup>1</sup> The University of Hong Kong,
<sup>2</sup> The University of Texas at Austin,
<sup>3</sup> The Chinese University of Hong Kong.



[[Paper](https://ieeexplore.ieee.org/document/11432899)]

This work is an extension of [AnalogCoder](https://arxiv.org/abs/2405.14918) (AAAI 2025) [[repo](https://github.com/laiyao1/AnalogCoder)].

# 🔔 Updates

- Our work has been accepted in **IEEE Transactions on Computer-Aided Design of Integrated Circuits and Systems (TCAD) 2026**! 🎉 


# 🎯 Overview

- **Challenge**: Analog front-end design still relies heavily on expert intuition and iterative simulations, with limited automation.  
- **Solution**: **AnalogCoder-Pro** — a unified multimodal LLM-based framework for analog design automation.  
- **Key Features**:  
  - Joint **circuit topology generation** and **device sizing optimization**  
  - Automatic generation of **performance-specific schematic netlists**  
  - **Multimodal diagnosis & repair** using specifications and waveform images  
  - Automated extraction of design parameters and parameter space formulation  
- **Outcome**: Improves design success rate and circuit performance, enabling an end-to-end automated workflow.  


# ✅ Project Checklist

- [x] Update the LLM run scripts.
- [x] Update the sample waveform figures.
- [ ] Update the BO optimization.
- [ ] Update all ablation study prompts.

# 🗂️ Software Structure

## Overall Workflow

```
User specifies task_id → run.py reads problem_set.tsv
       ↓
LLM generates PySpice circuit code (prompt_template*.md)
       ↓
run.py executes the code → ngspice simulation
       ↓
Error?  ┌─ Execution error  → execution_error.md  → LLM retry
        ├─ Floating node    → simulation_error.md → LLM retry
        └─ Bad waveform     → vlm_debug_prompt.md → VLM retry (optional)
       ↓
Functional check (problem_check/<Type>.py)
       ↓
[Opt tasks only] optimize_template.md → LLM parameterises circuit
                                       → Bayesian optimisation (Optuna)
```

## Root-level Files

| File | Description |
|------|-------------|
| `run.py` | **Main entry point.** Parses CLI arguments, reads task specs from `problem_set.tsv`, calls the LLM to generate PySpice circuit code, runs ngspice simulation, handles execution/simulation errors via iterative LLM feedback, optionally invokes the VLM for waveform-based diagnosis, performs DC-sweep bias-point search, runs functional test benches, and orchestrates Bayesian optimisation for Opt-level tasks. |
| `opamp.py` | Defines the `Opamp` PySpice `SubCircuitFactory` — a 5 V CMOS differential op-amp used as a pre-built building block by complex circuit designs (oscillators, integrators, differentiators, Schmitt triggers, etc.). |
| `dc_sweep_template.py` | Template code that is injected into generated amplifier/op-amp circuits. Performs a three-level (coarse → medium → fine) DC sweep to automatically find the optimal input bias point where Vout ≈ VDD/2. |
| `prompt_template.md` | LLM prompt template for **standard MOSFET-level** circuit generation (5 V supply, 1 µm technology). Includes a worked example and design rules. |
| `prompt_template_comptex.md` | LLM prompt template for **complex circuits** that reuse the pre-built `Opamp` subcircuit (e.g., oscillators, integrators, adders, Schmitt triggers). Shows how to instantiate and wire the `Opamp` subcircuit. |
| `prompt_template_optimize.md` | LLM prompt template for **optimisation-mode** circuit generation (45 nm technology, 1.2 V supply). Instructs the LLM to import technology-specific MOSFET parameters from a `model` module. |
| `optimize_template.md` | Prompt template that asks the LLM to convert a fixed-parameter PySpice circuit into a **parameterised** `create_circuit(params)` function and define Optuna search ranges, enabling automated Bayesian optimisation. |
| `execution_error.md` | Error-recovery prompt template. When generated circuit code raises a Python or structural error, this prompt (with the error message filled in) is sent back to the LLM for correction. |
| `simulation_error.md` | Error-recovery prompt template for **floating-node** ngspice errors. Identifies the floating node and asks the LLM to reconnect it. |
| `vlm_debug_prompt.md` | Prompt template for the **vision-language model (VLM)** component. Given a waveform image and expected behaviour description, the VLM analyses the waveform and explains discrepancies (no circuit suggestions — diagnosis only). |
| `problem_set.tsv` | **Benchmark dataset.** Defines 29 standard circuit-design tasks (IDs 1–29, levels Easy/Medium/Hard) and 16 optimisation tasks (IDs 51–66). Columns: `Id`, `Level`, `Circuit` description, `Input`/`Output` node names, `Type`, `Submodule Name`, `Testbench` instructions, `Normal` output description. |

## `problem_check/` Directory

Contains functional **test-bench scripts** for each circuit type. After a circuit passes simulation, `run.py` appends the corresponding test bench to the generated code and re-executes it to verify correct electrical behaviour.

| File | Circuit Type Tested |
|------|---------------------|
| `Amplifier.py` | Single/multi-stage amplifiers — measures AC voltage gain at 100 Hz |
| `Opamp.py` | Differential op-amps — measures differential-mode AC gain |
| `Inverter.py` | MOSFET inverters — checks voltage inversion |
| `CurrentMirror.py` | Current mirrors — verifies current mirroring ratio |
| `Comparator.py` | Comparators — sweeps Vin 0→5 V and checks output switching |
| `LowPass.py` | Low-pass filters — frequency sweep, checks roll-off |
| `HighPass.py` | High-pass filters — frequency sweep, checks pass-band |
| `BandPass.py` | Band-pass filters — frequency sweep, checks centre-band gain |
| `BandStop.py` | Band-stop filters — frequency sweep, checks notch attenuation |
| `Oscillator.py` | RC / Wien-bridge oscillators — transient sim, checks periodicity |
| `OscillatorFFT.py` | Oscillators with FFT — verifies dominant frequency component |
| `Integrator.py` | Op-amp integrators — inputs square wave, checks triangle output |
| `Differentiator.py` | Op-amp differentiators — inputs triangle wave, checks square output |
| `Adder.py` | Op-amp adders — verifies Vout = −(Vin1 + Vin2) |
| `Subtractor.py` | Op-amp subtractors — verifies Vout = Vin2 − Vin1 |
| `Mixer.py` | Gilbert cell mixers — FFT checks down/up-conversion products |
| `Schmitt.py` | Schmitt triggers — inputs sine wave, checks square-wave output |

## `sample_design/` Directory

Contains **reference circuit implementations** for all benchmark tasks (p1–p28). Each subdirectory `pN/` holds:

- `pN.py` — A working PySpice circuit file for task N.
- `pN_waveform.png` *(where present)* — A simulation waveform image demonstrating correct circuit behaviour.
- `opamp.py` *(in folders p9, p22–p28)* — A local copy of the op-amp subcircuit required by that design.

# 🧪 Benchmark
- Task descriptions are in the file `problem_set.tsv`.
- Sample circuits are in the `sample_design` directory.
- Test benches are in the `problem_check` directory.

# Environment Settings

```
git clone git@github.com:laiyao1/AnalogCoderPro.git
cd AnalogCoderPro
conda create -n analog python==3.10
conda activate analog
pip install matplotlib pandas numpy scipy openai
conda install -c conda-forge ngspice -y
conda install -c conda-forge pyspice
```

# Quick Start
```
python run.py --task_id=19 --num_per_task=3  --model=gpt-5-mini --api_key="[API_KEY]" --base_url="[BASE_URL]"
```
This script will attempt Mixer generation 3 times.
The mapping of task IDs can be found in `problem_set.tsv`.

# 📊 Waveform Gallery

Here are example waveforms for different circuit types, demonstrating the appropriate analysis methods for each design.

<table>
<colgroup><col style="width:50%"><col style="width:50%"></colgroup>
<tr>
<td align="center"><strong>Mixer</strong> — Transient + FFT Spectrum</td>
<td align="center"><strong>Schmitt Trigger</strong> — Transient + DC Transfer</td>
</tr>
<tr>
<td align="center"><img src="sample_design/p19/p19_waveform.png" alt="Mixer" style="width:95%; border-radius:8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1);"></td>
<td align="center"><img src="sample_design/p28/p28_waveform.png" alt="Schmitt Trigger" style="width:95%; border-radius:8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1);"></td>
</tr>
<tr>
<td align="center"><strong>Oscillator</strong> — Transient</td>
<td align="center"><strong>Integrator</strong> — Transient</td>
</tr>
<tr>
<td align="center"><img src="sample_design/p22/p22_waveform.png" alt="Oscillator" style="width:95%; border-radius:8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1);"></td>
<td align="center"><img src="sample_design/p24/p24_waveform.png" alt="Integrator" style="width:95%; border-radius:8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1);"></td>
</tr>
<tr>
<td align="center"><strong>Differentiator</strong> — Transient</td>
<td align="center"><strong>BandStop Filter</strong> — AC</td>
</tr>
<tr>
<td align="center"><img src="sample_design/p25/p25_waveform.png" alt="Differentiator" style="width:95%; border-radius:8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1);"></td>
<td align="center"><img src="sample_design/p13/p13_waveform.png" alt="BandStop Filter" style="width:95%; border-radius:8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1);"></td>
</tr>
<tr>
<td align="center"><strong>Comparator</strong> — DC Sweep</td>
<td align="center"></td>
</tr>
<tr>
<td align="center"><img src="sample_design/p9/p9_waveform.png" alt="Comparator" style="width:95%; border-radius:8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1);"></td>
<td align="center"></td>
</tr>
</table>


# 📚 Citation
If you find our work useful, we would appreciate a citation of our paper.


```

@article{lai2025analogcoder,
    title={AnalogCoder: Analog Circuit Design via Training-Free Code Generation},
    volume={39},
    DOI={10.1609/aaai.v39i1.32016},
    number={1},
    journal={Proceedings of the AAAI Conference on Artificial Intelligence},
    author={Lai, Yao and Lee, Sungyoung and Chen, Guojin and Poddar, Souradip and Hu, Mengkang and Pan, David Z. and Luo, Ping},
    year={2025},
    pages={379-387}
}

@article{lai2026analogcoderpro,
    author={Lai, Yao and Poddar, Souradip and Lee, Sungyoung and Chen, Guojin and Hu, Mengkang and Yu, Bei and Luo, Ping and Pan, David Z.},
    journal={IEEE Transactions on Computer-Aided Design of Integrated Circuits and Systems}, 
    title={AnalogCoder-Pro: Unifying Analog Circuit Generation and Optimization via Multi-modal LLMs}, 
    year={2026},
    doi={10.1109/TCAD.2026.3673493}
}

```
