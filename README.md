# Evo2 on Lawrencium

This guide helps you run Evo2 on the Lawrencium cluster using a pre-built container image (`evo2.sif`) or build it from the provided `evo2.def` definition file.

## Pre-Built Container Image Available

A pre-built `evo2.sif` image is available in the `/global/scratch/users/fengchenliu/JGI/GenomeOcean/` directory. If you are looking to quickly run Evo2 without building the image, you can use this pre-built container and follow the example steps below to run the same tests as before.

### Test the Pre-Built Container Image

If you want to test the pre-built container image (`evo2.sif`), simply run the following commands:

```bash
# Clone this repository
git clone git@github.com:jgi-genomeocean/evo2-on-lrc.git
cd evo2-on-lrc

# Clone the Evo2 repository
git clone --recurse-submodules https://github.com/ArcInstitute/evo2.git

cp ./test_evo2_fp16.py ./evo2/test
cp ./test_evo2_generation_fp16.py ./evo2/test

cd evo2

# Run tests with the pre-built container on A40 GPUs
apptainer exec --nv /global/scratch/users/fengchenliu/JGI/GenomeOcean/evo2.sif python ./test/test_evo2_fp16.py
apptainer exec --nv /global/scratch/users/fengchenliu/JGI/GenomeOcean/evo2.sif python ./test/test_evo2_generation_fp16.py

# Run tests with the pre-built container on H100 GPUs
apptainer exec --nv /global/scratch/users/fengchenliu/JGI/GenomeOcean/evo2.sif python ./test/test_evo2.py
apptainer exec --nv /global/scratch/users/fengchenliu/JGI/GenomeOcean/evo2.sif python ./test/test_evo2_generation.py
```

This will allow you to run the same tests and examples as previously, without needing to build the container yourself.

---

## Building the Container Image (Optional)

If you want to build the container image yourself, follow the steps below.

### Prerequisites

- Apptainer installed on your system (for container build and execution).
- Access to the Lawrencium cluster.
- NVIDIA GPUs (A40, A100, H100, etc.).
- Ensure you have enough space in `/global/scratch/users/$USER/`.

### Set Up Apptainer Cache Directory

Before proceeding, ensure the Apptainer cache directory is set to a location with sufficient space:

```bash
mkdir -p /global/scratch/users/$USER/.apptainer
export APPTAINER_CACHEDIR=/global/scratch/users/$USER/.apptainer
```

### 1. Create the Evo2 Definition File

Create a file called `evo2.def` with the following contents:

```bash
Bootstrap: docker
From: nvidia/cuda:12.1.0-cudnn8-devel-ubuntu22.04

%post
    # Set non-interactive mode to avoid prompts during installation
    export DEBIAN_FRONTEND=noninteractive
    export TZ=Etc/UTC

    # Update package list and install essential tools
    apt-get update && apt-get install -y \
        software-properties-common \
        git \
        curl \
        wget \
        build-essential \
        python3.11 \
        python3.11-venv \
        python3.11-dev

    # Create and activate a virtual environment
    python3.11 -m venv /opt/evo2_env
    . /opt/evo2_env/bin/activate

    # Install pip
    curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
    python get-pip.py
    /opt/evo2_env/bin/pip install --upgrade setuptools

    # Install Python dependencies
    /opt/evo2_env/bin/pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
    /opt/evo2_env/bin/pip install ninja cmake pybind11 numpy psutil biopython huggingface_hub

    # Clone and install Evo2
    git clone --recurse-submodules https://github.com/ArcInstitute/evo2.git /opt/evo2
    cd /opt/evo2
    /opt/evo2_env/bin/pip install .

%environment
    # Activate the virtual environment on container start
    . /opt/evo2_env/bin/activate

%runscript
    echo "Evo2 container is ready."
```

### 2. Build the Container Image

Once the `evo2.def` file is ready, build the container image using the following commands:

```bash
apptainer build evo2.sif evo2.def
```

### 3. Test the Container Image

After building the image, you can run the tests to make sure Evo2 works correctly on the container:

```bash
git clone --recurse-submodules https://github.com/ArcInstitute/evo2.git
cd evo2
apptainer exec --nv ../evo2.sif python ./test/test_evo2.py
apptainer exec --nv ../evo2.sif python ./test/test_evo2_generation.py
```

### 4. Optional: Use FP16 Test Files for Better Performance

For better performance on A40 and A100 GPUs, you can use the FP16 versions of the test files. Find them here:
* [test_evo2_fp16.py](test_evo2_fp16.py)
* [test_evo2_generation_fp16.py](test_evo2_generation_fp16.py)

### 5. Run Evo2 on the Cluster

Once you’ve tested everything, you can run Evo2 on the cluster using Apptainer:

```bash
apptainer exec --nv evo2.sif python your_script.py
```

### Notes

- Make sure to monitor GPU usage if running on multiple GPUs.
- The container has been tested on nodes with 8 H100 GPUs, 4 A100 GPUs, and 4 A40 GPUs, but you can scale it to your specific hardware configuration.

---

If you run into any issues, make sure the `APPTAINER_CACHEDIR` is set properly to avoid space issues.
