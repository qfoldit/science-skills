# BioNeMo NIM API Reference (self-hosted)

Every endpoint below was verified against NVIDIA's own official
documentation during this session. All are **self-hosted Docker
containers** (`docker run ... -p 8000:8000 nvcr.io/nim/...`), each
requiring an `NGC_API_KEY` and, for the larger models, a specific GPU
class. None of this is the shared multi-tenant `integrate.api.nvidia.com`
hosted endpoint — that path prefix (`/v1/biology/<org>/<model>/...`) was
only found in a third-party OpenAPI aggregation in this session, not
NVIDIA's own docs pages, and should be independently verified before
relying on it.

## AlphaFold2

- Docs: <https://docs.nvidia.com/nim/bionemo/alphafold2/latest/endpoints.html>
- Image: `nvcr.io/nim/deepmind/alphafold2:<version>`
- Endpoint: `POST /protein-structure/alphafold2/predict-structure-from-sequence`
- Request: `{"sequence": str, "databases": ["uniref90","mgnify","small_bfd"], "algorithm": "mmseqs2"|"jackhmmer", "relax_prediction": bool}`
- Response: `{"pdbs": [<PDB text>, ...]}`
- Citation if used: Jumper, J., Evans, R., Pritzel, A. et al. "Highly
  accurate protein structure prediction with AlphaFold." Nature 596,
  583–589 (2021). <https://doi.org/10.1038/s41586-021-03819-2>

## DiffDock

- Docs: <https://docs.nvidia.com/nim/bionemo/diffdock/latest/advanced-usage.html>
- Image: `nvcr.io/nim/mit/diffdock:<version>`
- Endpoint: `POST /molecular-docking/diffdock/generate`
- Request: `{"protein": <PDB text>, "ligand": <SMILES|SDF|Mol2>, "ligand_file_type": "smiles"|"sdf"|"mol2", "num_poses": int, "time_divisions": int, "steps": int, "save_trajectory": bool, "is_staged": bool}`
- Response: `{"ligand_positions": [...], "position_confidence": [...]}`
- Citation: Corso, G., Stärk, H., Jing, B., Barzilay, R., Jaakkola, T.
  "DiffDock: Diffusion Steps, Twists, and Turns for Molecular Docking."

## MolMIM

- Docs: <https://docs.nvidia.com/nim/bionemo/molmim/latest/endpoints.html>
- Image: `nvcr.io/nim/nvidia/molmim:<version>`
- Endpoints:
  - `POST /hidden` — `{"sequences": [<SMILES>]}` → latent code
  - `POST /decode` — decode a hidden-state representation back to SMILES
  - `POST /sampling` — `{"sequences": [<seed SMILES>], ...}` → unguided samples around the seed
  - `POST /generate` — CMA-ES-guided optimization against a scoring function (payload not independently verified in this session — check `/docs` on your running container before relying on exact field names)
  - `POST /embedding` — `{"sequences": [<SMILES>]}` → fixed-size embedding

## Evo 2

- Docs: <https://docs.nvidia.com/nim/bionemo/evo2/latest/endpoints.html>
- Image: `nvcr.io/nim/arc/evo2:<version>` (defaults to the 40B checkpoint)
- **Requires H100/H200-class GPU(s)** (FP8 support) — older GPUs unsupported.
- Endpoint: `POST /biology/arc/evo2/generate`
- Request: `{"sequence": <DNA string A/T/G/C>, "num_tokens": int, "temperature": float, "top_k": int, "top_p": float, "enable_sampled_probs": bool}`
- Response: `{"sequence": <generated nucleotides>, ...}`
- Model: Hyena-based genomic foundation model, up to 1M-token context,
  trained on 11 trillion DNA nucleotides.

## Boltz-2

- Docs: <https://docs.nvidia.com/nim/bionemo/boltz2/latest/inference.html>
- Image: `nvcr.io/nim/mit/boltz2:<version>`
- **Requires >=48GB VRAM GPU.**
- Endpoint: `POST /biology/mit/boltz2/predict`
- Request: `{"polymers": [{"id","molecule_type","sequence",...}], "ligands": [{"id","smiles"|"ccd"}], "constraints": [...], "recycling_steps": int, "sampling_steps": int, "diffusion_samples": int, "step_scale": float, "output_format": "mmcif"|...}`
- Response: `{"structures": [{"format","structure"}], "confidence_scores": [...]}`
- Citation: Passaro et al. 2025, "Boltz-2: Towards Accurate and Efficient
  Binding Affinity Prediction."
- Max 4096 residues/chain, 12 polymers, 20 ligands per request.

## OpenFold3

- Docs: <https://docs.nvidia.com/nim/bionemo/openfold3/latest>
- Image: `nvcr.io/nim/openfold/openfold3:<version>`
- **Requires an NVIDIA GPU with enough VRAM for the target complex size**
  (A100 40/80GB recommended per NVIDIA's own docs).
- Endpoint: `POST /biology/openfold/openfold3/predict`
- Request: `{"inputs": [<complex_spec>]}` — **only the top-level shape and
  the "at most one complex per call" constraint were confirmed** against
  NVIDIA's docs in this session; the full field-by-field schema of
  `complex_spec` (entity type tags, ligand format, MSA/template fields)
  was NOT independently verified against a fetched worked example — fetch
  `docs.nvidia.com/nim/bionemo/openfold3/latest/example-requests.html`
  before building a real request.
- Model: OpenFold Consortium / AlQuraishi Lab's open PyTorch
  re-implementation of AlphaFold3, predicting complexes of proteins, DNA,
  RNA, and small-molecule ligands together (not single chains only).
- Citation: Abramson et al. 2024, *Nature* — cite the original AlphaFold3
  paper for the underlying architecture, not NVIDIA or this skill.

## Not implemented here

ESMFold, RFdiffusion, and ProteinMPNN — their NIM request/response
schemas were not independently verified in this session as NVIDIA-hosted
clients. **Note:** these three models ARE implemented, tested, and
runnable locally/via Docker (not via NVIDIA NIM) by this repository's
root-level `protein_design_mcp` MCP server
(`src/protein_design_mcp/pipelines/{esmfold,rfdiffusion,proteinmpnn}.py`,
with matching tests in `tests/test_{esmfold,rfdiffusion,proteinmpnn}.py`)
— prefer that server if you need those three models and can run local/
Docker compute, rather than waiting for a NIM-client version here. Extend
`bionemo_client.py` following the same pattern as the models above if a
lightweight NIM-hosted alternative is specifically wanted (confirm the
exact endpoint path and payload shape against
`docs.nvidia.com/nim/bionemo/<model>/latest/endpoints.html` or the
container's own `/docs` OpenAPI page before writing the wrapper — do not
guess).
