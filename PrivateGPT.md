# Setting up PrivateGPT on Windows

## Prerequisites
1. **Download and install Miniconda**: [https://docs.anaconda.com/miniconda/](https://docs.anaconda.com/miniconda/)
2. **Ensure Ollama path is added to environment variables**:
   - Follow instructions to add the Ollama path to environment variables.
   - Restart PowerShell if required.
3. **Follow detailed instructions**: [PrivateGPT Installation Guide](https://simplifyai.in/2023/11/privategpt-installation-guide-for-windows-machine-pc/)

## Steps to Install and Setup

1. **Clone the PrivateGPT Repository**:
   ```bash
   git clone https://github.com/imartinez/privateGPT
   cd privateGPT
   ```

2. **Create a Conda Environment**:
   ```bash
   conda create -n privateGPT python=3.11
   conda activate privateGPT
   ```

3. **Verify Python Version**:
   ```bash
   conda info
   ```
   If the Python version is incorrect, install the correct version:
   ```bash
   conda install -c conda-forge python=3.11
   ```

4. **Install Pipx and Poetry**:
   ```bash
   conda install -c conda-forge pipx
   pipx install poetry
   ```

5. **Install Dependencies Using Poetry**:
   ```bash
   poetry install --extras "ui llms-ollama embeddings-ollama vector-stores-qdrant llms-llama-cpp"
   ```

6. **Set Environment Variable for PGPT Profiles**:
   ```bash
   $env:PGPT_PROFILES="ollama"
   ```

7. **Run the Application**:
   - For local usage:
     ```bash
     poetry run python -m uvicorn private_gpt.main:app --reload --port 8001
     ```
   - To run it over a network:
     ```bash
     poetry run python -m uvicorn private_gpt.main:app --reload --port 8001 --host 0.0.0.0
     ```

## GPU Setup Commands (If you have GPU, you are recommended to perform below provided steps)

1. **Install PyTorch with CUDA Support**:
   ```bash
   pip install torch==2.0.0+cu118 --index-url https://download.pytorch.org/whl/cu118
   ```

2. **Enable CUDA Support for llama-cpp-python**:
   ```bash
   $env:CMAKE_ARGS='-DGGML_CUDA=on'
   poetry run pip install --force-reinstall --no-cache-dir llama-cpp-python==0.2.32
   ```

3. **Install Compatible NumPy Version**:
   ```bash
   poetry run pip install numpy==1.24.3
   ```

4. **Enable LLAMA CUBLAS Support**:
   ```bash
   $env:CMAKE_ARGS='-DLLAMA_CUBLAS=on'
   poetry run pip install --force-reinstall --no-cache-dir llama-cpp-python numpy==1.26.0
   ```

---

By following these steps, you can successfully set up PrivateGPT on a Windows machine with optional GPU acceleration.

