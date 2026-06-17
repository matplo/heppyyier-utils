# heppyyier-utils

Workflow utilities for HEP event generators and tools installed with
[heppyyier](https://github.com/matplo/heppyyier).

The repository is organized by generator/tool namespace. JEWEL utilities live
under `heppyyier_utils.jewel`, and all JEWEL command-line entry points use a
`jewel_` prefix so future utilities for other generators can coexist cleanly.

## JEWEL

V1 provides helpers for staging, running, and converting JEWEL event generation.
The utilities are split into four commands:

- `jewel_prepare`: create self-contained run directories and patched JEWEL
  parameter files. It does not run JEWEL.
- `jewel_run`: run JEWEL inside directories previously created by
  `jewel_prepare`.
- `jewel_convert`: convert one HepMC file to a CAB-compatible ROOT file using
  `pyhepmc` and `uproot`.
- `jewel_pipeline`: do `prepare` and `run` in one command, with optional
  conversion.

Install the utility package:

```bash
heppyyier recipe update
heppyyier install heppyyier-utils
module load heppyyier-utils
```

`heppyyier-utils` itself does not hard-depend on JEWEL. This keeps the package
installable for future non-JEWEL utilities; only the `jewel_` commands require
JEWEL/LHAPDF at runtime.

For JEWEL production, install/load the generator stack separately:

```bash
heppyyier install jewel lhapdf
module load jewel lhapdf
```

The required LHAPDF sets for the bundled JEWEL templates are:

- pp/vacuum: `CT14nlo` (`PDFSET 13100`)
- PbPb/medium: `EPPS16nlo_CT14nlo_Pb208` (`PDFSET 901300`)

Install the PDF sets through LHAPDF if they are not already present:

```bash
lhapdf install CT14nlo
lhapdf install EPPS16nlo_CT14nlo_Pb208
```

## JEWEL Workflows

Prepare both PbPb and pp run directories without running JEWEL:

```bash
jewel_prepare --tag pt100 --samples both --nevents 10000
```

Run JEWEL from the generated manifests:

```bash
jewel_run outputs/jewel_events/pt100
```

Convert one HepMC file to the ROOT tree format expected by CAB analysis:

```bash
jewel_convert events.hepmc events.root
```

For medium/PbPb JEWEL output with recoil subtraction enabled in the JEWEL
params, use 4-momentum subtraction during conversion:

```bash
jewel_convert jewel_med.hepmc jewel_med.root --subtract-4mom
```

Run prepare, JEWEL execution, and conversion in one command:

```bash
jewel_pipeline --tag pt100 --samples both --nevents 10000 --convert
```

Run one sample at a time:

```bash
jewel_prepare --tag pbpb_001 --samples medium --job-id 1
jewel_prepare --tag pp_001 --samples vacuum --job-id 1
```

Conversion reads HepMC with `pyhepmc` and writes CAB-compatible ROOT TTrees with
`uproot`. It does not use PyROOT or native ROOT libraries.

## Run Directory Layout

For `--tag pt100 --samples both`, `jewel_prepare` writes:

```text
outputs/jewel_events/pt100/
  jewel_med/
    params.dat
    medium.params.dat
    manifest.yaml
    events/jewel_med_pt100.hepmc
    logs/jewel_med_pt100.log
    logs/jewel_med_pt100.stdout.log
    roots/jewel_med_pt100.root
    splitint/jewel_med_pt100.dat
    xsecs/jewel_med_pt100.dat
  jewel_vac/
    params.dat
    manifest.yaml
    events/jewel_vac_pt100.hepmc
    logs/jewel_vac_pt100.log
    logs/jewel_vac_pt100.stdout.log
    roots/jewel_vac_pt100.root
    splitint/jewel_vac_pt100.dat
    xsecs/jewel_vac_pt100.dat
```

`params.dat` is the patched JEWEL parameter file. `manifest.yaml` records the
executable, parameter file, HepMC output, ROOT output, logs, and PDF set names.
`jewel_run` reads this manifest instead of re-deriving paths.

## Command Help

`jewel_prepare --help`:

```text
Usage: python -m heppyyier_utils.jewel.cli prepare [OPTIONS]

  Prepare self-contained JEWEL run directories.

Options:
  --samples TEXT            Samples to prepare: both, medium/PbPb, or
                            vacuum/pp.  [default: both]
  --out-dir TEXT            Output directory.  [default: outputs/jewel_events]
  --tag TEXT                Run tag. Default: current timestamp.
  --clean                   Remove selected tag/sample run directories before
                            preparing.
  --nevents TEXT            Override NEVENT for both samples.
  --nevents-medium TEXT     Override NEVENT for the medium/PbPb sample.
  --nevents-vacuum TEXT     Override NEVENT for the vacuum/pp sample.
  --ptmin TEXT              Override PTMIN for both samples.
  --ptmax TEXT              Override PTMAX for both samples.
  --etamax TEXT             Set or override ETAMAX for both samples.
  --job-id TEXT             Override NJOB for both samples.
  --job-id-medium TEXT      Override NJOB for the medium/PbPb sample.
  --job-id-vacuum TEXT      Override NJOB for the vacuum/pp sample.
  --medium-bin TEXT         Medium JEWEL executable.  [default:
                            jewel-2.4.0-simple]
  --vacuum-bin TEXT         Vacuum JEWEL executable.  [default:
                            jewel-2.4.0-vac]
  --template-dir DIRECTORY  Directory with JEWEL template .dat files.
  -h, --help                Show this message and exit.
```

`jewel_run --help`:

```text
Usage: python -m heppyyier_utils.jewel.cli run [OPTIONS] RUN_PATH

  Run prepared JEWEL sample directories from manifests.

Options:
  --samples TEXT     Samples to run when RUN_PATH is a tag/root directory.
                     [default: both]
  --no-output-check  Do not require a non-empty HepMC file after JEWEL exits.
  -h, --help         Show this message and exit.
```

`jewel_convert --help`:

```text
Usage: python -m heppyyier_utils.jewel.cli convert [OPTIONS] INPUT_HEPMC
                                                   OUTPUT_ROOT

  Convert HepMC to CAB-compatible ROOT TTrees using uproot.

Options:
  --subtract-4mom / --no-subtract-4mom
                                  Apply JEWEL 4-momentum recoil subtraction.
                                  [default: no-subtract-4mom]
  --max-events INTEGER            Maximum events to convert.
  --final-status INTEGER          HepMC status used for final particles.
                                  [default: 1]
  --ghost-status INTEGER          HepMC status used for JEWEL thermal ghosts.
                                  [default: 3]
  --progress / --no-progress      Show a tqdm conversion progress bar.
                                  [default: progress]
  --event-info / --no-event-info  Write the event_info tree.  [default: event-
                                  info]
  -h, --help                      Show this message and exit.
```

`jewel_pipeline --help`:

```text
Usage: python -m heppyyier_utils.jewel.cli pipeline [OPTIONS]

  Prepare and run JEWEL, optionally converting HepMC to ROOT.

Options:
  --samples TEXT                  Samples to prepare: both, medium/PbPb, or
                                  vacuum/pp.  [default: both]
  --out-dir TEXT                  Output directory.  [default:
                                  outputs/jewel_events]
  --tag TEXT                      Run tag. Default: current timestamp.
  --clean                         Remove selected tag/sample run directories
                                  before preparing.
  --nevents TEXT                  Override NEVENT for both samples.
  --nevents-medium TEXT           Override NEVENT for the medium/PbPb sample.
  --nevents-vacuum TEXT           Override NEVENT for the vacuum/pp sample.
  --ptmin TEXT                    Override PTMIN for both samples.
  --ptmax TEXT                    Override PTMAX for both samples.
  --etamax TEXT                   Set or override ETAMAX for both samples.
  --job-id TEXT                   Override NJOB for both samples.
  --job-id-medium TEXT            Override NJOB for the medium/PbPb sample.
  --job-id-vacuum TEXT            Override NJOB for the vacuum/pp sample.
  --medium-bin TEXT               Medium JEWEL executable.  [default:
                                  jewel-2.4.0-simple]
  --vacuum-bin TEXT               Vacuum JEWEL executable.  [default:
                                  jewel-2.4.0-vac]
  --template-dir DIRECTORY        Directory with JEWEL template .dat files.
  --convert / --no-convert        Convert generated HepMC files to ROOT after
                                  running.  [default: no-convert]
  --subtract-4mom-medium / --no-subtract-4mom-medium
                                  Apply 4-momentum subtraction when converting
                                  medium/PbPb samples.  [default:
                                  subtract-4mom-medium]
  --no-output-check               Do not require a non-empty HepMC file after
                                  JEWEL exits.
  -h, --help                      Show this message and exit.
```

## Development

```bash
python -m pip install -e '.[test]'
python -m pytest
```
