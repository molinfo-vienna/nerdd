FROM continuumio/miniconda3:4.10.3p0-alpine

# Necessary, so Docker doesn't buffer the output and that we can see the output 
# of the application in real-time.
ENV PYTHONUNBUFFERED 1

WORKDIR /app

#
# Setup the conda environment
# 1. create the conda environment
# 2. get the name of the conda environment and write to bashrc
# 3. configure auto-activation of the conda environment (on each RUN command)
#

# 1. create the conda environment
COPY environment.yml .
RUN conda env create -f environment.yml 

# Copy the nerdd-module source code directory and install it
# --mount=type=cache,target=/root/.cache/pip: create a cache directory for pip, so that
# it doesn't have to download the dependencies every time we build the image
COPY --from=nerdd-module . /deps/nerdd-module
RUN --mount=type=cache,target=/root/.cache/pip \
    conda run -n nerdd_backend pip install -e /deps/nerdd-module

# Copy the nerdd-link source code directory and install it
COPY --from=nerdd-link . /deps/nerdd-link
RUN --mount=type=cache,target=/root/.cache/pip \
    conda run -n nerdd_backend pip install -e /deps/nerdd-link

# 2. get the name of the created conda environment
# ... and write it as an environment variable to the ~/.bashrc file
# RUN echo "export CONDA_DEFAULT_ENV=$(head -1 environment.yml | cut -d' ' -f2)" >> ~/.bashrc

# # 3. activate the conda environment
# RUN echo "conda activate \$CONDA_DEFAULT_ENV" >> ~/.bashrc
# RUN echo "export PATH=/opt/conda/envs/\$CONDA_DEFAULT_ENV/bin:\$PATH" >> ~/.bashrc

COPY . .

# --mount=type=cache,target=/root/.cache/pip: create a cache directory for pip, so that
# it doesn't have to download the dependencies every time we build the image
RUN --mount=type=cache,target=/root/.cache/pip \
    conda run -n nerdd_backend pip install -e .[test]

# --no-capture-output: output of the application is not buffered and we can see it 
# Note: PYTHONUNBUFFERED is not enough, because it only affects the Python
# standard output, not the output of the application.
ENTRYPOINT conda run --no-capture-output -n nerdd_backend \
    python -m nerdd_backend.main