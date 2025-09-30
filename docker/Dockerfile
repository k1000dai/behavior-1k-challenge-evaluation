FROM nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04

RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    python3-venv \
    git vim \
    && rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates
RUN apt-get update && apt-get install -y \
    libgl1 \
    libglib2.0-0 \
    libsm6 \
    libxext6 \
    libxrender1 \
    libglvnd0 \
    libegl1 \
    ffmpeg \
    && rm -rf /var/lib/apt/lists/*
RUN apt-get update && apt-get install -y \
    libxt6 \
    libglu1-mesa \
    libxmu6 \
    libxi6 \
    libxrender1 \
    libxext6 \
    libx11-6 \
    libxrandr2 \
    libxcursor1 \
    libxinerama1 \
    libglib2.0-0 \
    libfontconfig1 \
    && rm -rf /var/lib/apt/lists/*
ADD https://astral.sh/uv/install.sh /uv-installer.sh
RUN sh /uv-installer.sh && rm /uv-installer.sh
ENV PATH="/root/.local/bin/:$PATH"

# Build-time options
ARG BEHAVIOR_BRANCH=main
ARG CUDA_VERSION=12.4
ARG INSTALL_BDDL=true
ARG INSTALL_OMNIGIBSON=true
ARG INSTALL_JOYLO=true
ARG INSTALL_PRIMITIVES=false
ARG INSTALL_EVAL=true
ARG INSTALL_DEV=false
ARG INSTALL_DATASET=true
ARG ACCEPT_NVIDIA_EULA=true
ARG ACCEPT_DATASET_TOS=true

# Clone BEHAVIOR-1K repo
WORKDIR /opt
RUN git clone -b ${BEHAVIOR_BRANCH} https://github.com/StanfordVL/BEHAVIOR-1K.git behavior-1k

WORKDIR /opt/behavior-1k

# Create uv venv with Python 3.10 and install base deps + PyTorch CUDA wheels
RUN set -e; \
    if command -v python3.10 >/dev/null; then PYBIN=python3.10; else PYBIN=python3; fi; \
    uv venv --python "$PYBIN" .venv; \
    . .venv/bin/activate; \
    uv pip install "numpy<2" "setuptools<=79"; \
    CUDA_VER_SHORT=$(echo ${CUDA_VERSION} | sed 's/\\.//g'); \
    uv pip install pip; \
    uv pip install torch torchvision torchaudio --torch-backend=auto; \
    uv pip install hydra-core gspread pandas av msgpack

# Make venv active for subsequent layers
ENV VIRTUAL_ENV=/opt/behavior-1k/.venv
ENV PATH="/opt/behavior-1k/.venv/bin:$PATH"

# Install BDDL 
RUN if [ "$INSTALL_BDDL" = "true" ]; then \
      [ -d bddl ] || { echo "bddl directory missing"; exit 1; }; \
      uv pip install -e bddl; \
    fi

# Install OmniGibson (editable) with optional extras
RUN if [ "$INSTALL_OMNIGIBSON" = "true" ]; then \
      [ -d OmniGibson ] || { echo "OmniGibson directory missing"; exit 1; }; \
      PY_VER=$(python -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")'); \
      [ "$PY_VER" = "3.10" ] || { echo "Python 3.10 required for OmniGibson (found ${PY_VER})"; exit 1; }; \
      if [ -n "${EXP_PATH}" ] || [ -n "${CARB_APP_PATH}" ] || [ -n "${ISAAC_PATH}" ]; then \
        echo "ERROR: Found existing Isaac Sim environment variables"; exit 1; \
      fi; \
      EXTRAS=""; \
      [ "$INSTALL_DEV" = "true" ] && EXTRAS="${EXTRAS}dev,"; \
      [ "$INSTALL_PRIMITIVES" = "true" ] && EXTRAS="${EXTRAS}primitives,"; \
      [ "$INSTALL_EVAL" = "true" ] && EXTRAS="${EXTRAS}eval,"; \
      if [ -n "$EXTRAS" ]; then EXTRAS="[${EXTRAS%,}]"; fi; \
      uv pip install -e "OmniGibson${EXTRAS}"; \
      if [ "$INSTALL_DEV" = "true" ]; then \
        uv pip install pre-commit && (cd OmniGibson && pre-commit install); \
      fi; \
    fi

# NVIDIA EULA and Isaac Sim wheels
RUN set -e; \
    if [ "$INSTALL_OMNIGIBSON" = "true" ]; then \
      if [ "$ACCEPT_NVIDIA_EULA" != "true" ]; then echo "You must set --build-arg ACCEPT_NVIDIA_EULA=true"; exit 1; fi; \
      export OMNI_KIT_ACCEPT_EULA=YES; \
      if python -c "import isaacsim" >/dev/null 2>&1; then \
        echo "Isaac Sim already installed, skipping..."; \
      else \
        echo "Installing Isaac Sim wheels..."; \
        tmpdir=$(mktemp -d); cd "$tmpdir"; \
        packages="omniverse_kit-106.5.0.162521 isaacsim_kernel-4.5.0.0 isaacsim_app-4.5.0.0 \
          isaacsim_core-4.5.0.0 isaacsim_gui-4.5.0.0 isaacsim_utils-4.5.0.0 \
          isaacsim_storage-4.5.0.0 isaacsim_asset-4.5.0.0 isaacsim_sensor-4.5.0.0 \
          isaacsim_robot_motion-4.5.0.0 isaacsim_robot-4.5.0.0 isaacsim_benchmark-4.5.0.0 \
          isaacsim_code_editor-4.5.0.0 isaacsim_ros1-4.5.0.0 isaacsim_cortex-4.5.0.0 \
          isaacsim_example-4.5.0.0 isaacsim_replicator-4.5.0.0 isaacsim_rl-4.5.0.0 \
          isaacsim_robot_setup-4.5.0.0 isaacsim_ros2-4.5.0.0 isaacsim_template-4.5.0.0 \
          isaacsim_test-4.5.0.0 isaacsim-4.5.0.0 isaacsim_extscache_physics-4.5.0.0 \
          isaacsim_extscache_kit-4.5.0.0 isaacsim_extscache_kit_sdk-4.5.0.0"; \
        for pkg in $packages; do \
          pkg_name=${pkg%-*}; filename="${pkg}-cp310-none-manylinux_2_34_x86_64.whl"; \
          pkg_name_dash=$(echo "$pkg_name" | tr '_' '-'); \
          url="https://pypi.nvidia.com/${pkg_name_dash}/$filename"; \
          echo "Downloading $pkg"; curl -sSL "$url" -o "$filename"; \
        done; \
        uv pip install ./*.whl; \
        cd /opt/behavior-1k; rm -rf "$tmpdir"; \
        python -c "import isaacsim; print('isaacsim import ok')"; \
      fi; \
    fi

# Dataset (optional)
WORKDIR /opt
RUN set -e; \
    if [ "$INSTALL_OMNIGIBSON" = "true" ] && [ "$INSTALL_DATASET" = "true" ]; then \
      export OMNI_KIT_ACCEPT_EULA=YES; \
      python -c "from omnigibson.utils.asset_utils import download_omnigibson_robot_assets; download_omnigibson_robot_assets()"; \
      if [ "$ACCEPT_DATASET_TOS" = "true" ]; then PYFLAG=True; else PYFLAG=False; fi; \
      python -c "from omnigibson.utils.asset_utils import download_behavior_1k_assets; download_behavior_1k_assets(accept_license=${PYFLAG})"; \
      python -c "from omnigibson.utils.asset_utils import download_2025_challenge_task_instances; download_2025_challenge_task_instances()"; \
    fi


# JoyLo (optional)
WORKDIR /opt/behavior-1k
RUN if [ "$INSTALL_JOYLO" = "true" ]; then \
      [ -d joylo ] || { echo "joylo directory missing"; exit 1; }; \
      uv pip install -e joylo; \
    fi

# Fix cffi version to avoid potential issues with newer versions
RUN set -e; \
    uv pip install "cffi==1.17.1"

WORKDIR /workspace
