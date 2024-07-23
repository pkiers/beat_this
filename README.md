# Beat This!
Official implementation of the beat tracker from the ISMIR 2024 paper "Beat This! Accurate Beat Tracking Without DBN Postprocessing" by Francesco Foscarin, Jan Schlüter and Gerhard Widmer.

* [Inference](#inference)
* [Available models](#available-models)
* [Reproducing metrics from the paper](#reproducing-metrics-from-the-paper)
* [Training](#training)
* [Reusing the loss](#reusing-the-loss)
* [Reusing the model](#reusing-the-model)
* [Citation](#citation)


## Inference

To predict beats for audio files, you can either use our command line tool or call the beat tracker from Python. Both have the same requirements.

### Requirements

The beat tracker requires Python with a set of packages installed:
1. [Install PyTorch](https://pytorch.org/get-started/locally/) 2.0 or later following the instructions for your platform.
2. Install further modules with `pip install tqdm einops soxr rotary-embedding-torch`. (If using conda, we still recommend pip. You may try installing `soxr-python` and `einops` from conda-forge, but `rotary-embedding-torch` is only on PyPI.)
3. To read other audio formats than `.wav`, install `ffmpeg` or another supported backend for `torchaudio`. (`ffmpeg` can be installed via conda or via your operating system.)

Finally, install our beat tracker with:
```bash
pip install https://github.com/CPJKU/beat_this/archive/master.zip
```

### Command line

Along with the python package, a command line application called `beat_this` is installed. For a full documentation of the command line options, run:
```bash
beat_this --help
```
The basic usage is:
```bash
beat_this path/to/audio.file -o path/to/output.beats
```
To process multiple files, specify multiple input files or directories, and give an output directory instead:
```bash
beat_this path/to/*.mp3 path/to/whole_directory/ -o path/to/output_directory
```
The beat tracker will use the first GPU in your system by default, and fall back to CPU if PyTorch does not have CUDA access. With `--gpu=2`, it will use the third GPU, and with `--gpu=-1` it will force the CPU.
If you have a lot of files to process, you can distribute the load over multiple processes by running the same command multiple times with `--touch-first`, `--skip-existing` and potentially different options for `--gpu`:
```bash
for gpu in {0..3}; do beat_this input_dir -o output_dir --touch-first --skip-existing --gpu=$gpu &; done
```
If you want to use the DBN for postprocessing, add `--dbn`. The DBN parameters are the default ones from madmom. This requires installing the `madmom` package.

### Python class

If you are a Python user, you can directly use the `beat_this.inference` module.

First, instantiate an instance of the `File2Beats` class that encapsulates the model along with pre- and postprocessing:
```python
from beat_this.inference import File2Beats
file2beats = File2Beats(checkpoint_path="final0", device="cuda", dbn=False)
```
To obtain a list of beats and downbeats for an audio file, run:
```python
audio_path = "path/to/audio.file"
beats, downbeats = file2beats(audio_path)
```
Optionally, you can produce a `.beats` file (e.g., for importing into [Sonic Visualizer](https://www.sonicvisualiser.org/)):
```python
from beat_this.utils import save_beat_tsv
output_path = "path/to/output.beats"
save_beat_tsv(beats, downbeats, outpath)
```
If you already have an audio tensor loaded, instead of `File2Beats`, use `Audio2Beats` and pass the tensor and its sample rate. We also provide `Audio2Frames` for framewise logits and `Spect2Frames` for spectrogram inputs.


## Available models

Models are available for manual download at [our cloud space](https://cloud.cp.jku.at/index.php/s/7ik4RrBKTS273gp), but will also be downloaded automatically by the above inference code. By default, the inference will use `final0`, but it is possible to select another model via a command line option (`--model`) or Python parameter (`checkpoint_path`).

* `final0`, `final1`, `final2`: Our main model, which was trained on all data except the GTZAN dataset, with three different seeds. This corresponds to "Our system" in Table 2 of the paper. About 78 MiB per model.
* `small0`, `small1`, `small2`: A smaller model, again trained on all data except GTZAN, with three different seeds. This corresponds to "smaller model" in Table 2 of the paper. About 8.1 MiB per model.
* *More models will be added soon.*

Please be aware that, as the models `final*` and `small*` were trained on all data except the GTZAN dataset, the results may be unfairly good if you run inference on any file from the training datasets.

All the models are provided as PyTorch Lightning checkpoints, stripped of the optimizer state to reduce their size. This is useful for reproducing the paper results, or verifying the hyper parameters (stored in the checkpoint under `hyper_parameters` and `datamodule_hyper_parameters`).
During inference, PyTorch Lighting is not used, and the checkpoints are converted and loaded into vanilla PyTorch modules.


## Reproducing metrics from the paper

### Requirements

In addition to the [inference requirements](#requirements), computing evaluation metrics requires PyTorch Lightning and `mir_eval`, as well as obtaining and setting up the GTZAN dataset. *This part will be completed soon.*

### Command line

Compute results on the test set (GTZAN) corresponding to Table 2 in the paper.

Main results for our system:
```bash
python launch_scripts/compute_paper_metrics.py --models final0 final1 final2 --datasplit test
```

Smaller model:
```bash
python launch_scripts/compute_paper_metrics.py --models small0 small1 small2 --datasplit test
```

With DBN (this requires installing the madmom package):
```bash
python launch_scripts/compute_paper_metrics.py --models final0 final1 final2 --datasplit test --dbn
```

*This part will be completed soon.*


## Training

*This part will be available soon.*


## Reusing the loss

To reuse our shift-invariant binary cross-entropy loss, just copy out the `ShiftTolerantBCELoss` class from [`loss.py`](beat_this/model/loss.py), it does not have any dependencies.


## Reusing the model

To reuse the BeatThis model, you have multiple options:

### From the package

When installing the `beat_this` package, you can directly import the model class:
```
from beat_this.model.beat_tracker import BeatThis
```
Instantiating this class will give you an untrained model from spectrograms to frame-wise beat and downbeat logits. For a pretrained model, use `load_model`:
```
from beat_this.inference import load_model
beat_this = load_model('final0', device='cuda')
```
### From torch.hub

To quickly try the model without installing the package, just install the [requirements for inference](#requirements) and do:
```
import torch
beat_this = torch.hub.load('CPJKU/beat_this', 'beat_this', 'final0', device='cuda')
```
### Copy and paste

To copy the BeatThis model into your own project, you will need the [`beat_tracker.py`](beat_this/model/beat_tracker.py) and [`roformer.py`](beat/this/model/roformer.py) files. If you remove the `BeatThis.state_dict()` and `BeatThis._load_from_state_dict()` methods that serve as a workaround for compiled models, then there are no other internal dependencies, only external dependencies (`einops`, `rotary-embedding-torch`).


## Citation

```bibtex
@inproceedings{foscarin2024beatthis,
    author = {Francesco Foscarin and Jan Schl{\"u}ter and Gerhard Widmer},
    title = {Beat this! Accurate beat tracking without DBN postprocessing}
    year = 2024,
    month = nov,
    booktitle = {Proceedings of the 25th International Society for Music Information Retrieval Conference (ISMIR)},
    address = {San Francisco, CA, United States},
}
```
