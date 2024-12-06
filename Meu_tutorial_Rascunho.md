
# Instalação do Plugin Intel OpenVINO no Audacity 3.7 no Ubuntu 24.04

## Introdução
Este tutorial descreve como instalar o plugin Intel OpenVINO no Audacity 3.7 em um ambiente Ubuntu 24.04. 
Com o Audacity e o plugin Intel OpenVINO, você pode potencializar tarefas de edição e processamento de áudio com inteligência artificial, como transcrição de fala para texto, supressão de ruídos, separação de fontes (vocal/instrumentos), reconhecimento de emoções e idiomas, tradução automática, e aplicação de efeitos estilizados. Além disso, o OpenVINO permite processamento em tempo real, diagnóstico acústico avançado e otimização de desempenho em hardware Intel, tornando o Audacity uma ferramenta ainda mais poderosa para produção de áudio profissional ou amadora.

---

## Passo 1: Verifique sua GPU e o suporte ao OpenCL

1. Identifique a sua GPU:
    ```bash
    sudo lshw -C display
    ```

2. Instale os drivers adequados:
    - **GPU Intel:**
      ```bash
      sudo apt install -y intel-gpu-tools
      ```
    - **GPU AMD:**
      ```bash
      sudo apt install -y mesa-opencl-icd
      ```
    - **GPU NVIDIA:**
      ```bash
      sudo apt install -y nvidia-driver-535 nvidia-opencl-icd-535
      ```

3. Verifique o suporte ao OpenCL:
    ```bash
    sudo ldconfig
    clinfo
    ```

    Caso o resultado seja `Number of platforms = 0`, resolva os problemas de drivers antes de continuar.

---

## Passo 2: Instale as Dependências

```bash
sudo apt update
sudo apt install -y build-essential cmake git python3-pip libgtk2.0-dev libasound2-dev libjack-jackd2-dev uuid-dev ocl-icd-opencl-dev
sudo apt install -y python3.12-venv clinfo
```

---

## Passo 3: Baixe e Instale o OpenVINO

1. Baixe o pacote OpenVINO:
    ```bash
    mkdir ~/Downloads/openvino
    cd ~/Downloads/openvino
    wget -vc https://storage.openvinotoolkit.org/repositories/openvino/packages/2024.5/linux/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
    ```

2. Extraia e instale as dependências:
    ```bash
    tar xvf l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
    cd l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/install_dependencies/
    sudo -E ./install_openvino_dependencies.sh
    ```

3. Configure o ambiente:
    ```bash
    source ~/Downloads/openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/setupvars.sh
    echo "source ~/Downloads/openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/setupvars.sh" >> ~/.bashrc
    ```

---

## Passo 4: Baixe e Instale o Libtorch

1. Baixe e extraia o Libtorch:
    ```bash
    wget https://download.pytorch.org/libtorch/cpu/libtorch-cxx11-abi-shared-with-deps-2.4.1%2Bcpu.zip
    unzip libtorch-cxx11-abi-shared-with-deps-2.4.1+cpu.zip
    rm -f libtorch-cxx11-abi-shared-with-deps-2.4.1+cpu.zip
    export LIBTORCH_ROOTDIR=$(pwd)/libtorch
    ```

---

## Passo 5: Compile o Plugin OpenVINO

1. Clone o repositório do plugin:
    ```bash
    git clone https://github.com/intel/openvino-plugins-ai-audacity.git
    cd openvino-plugins-ai-audacity
    mkdir build
    cd build
    cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local
    make -j$(nproc)
    sudo make install
    ```

---

## Passo 6: Compile o Audacity com o Plugin OpenVINO

1. Clone o repositório do Audacity:
    ```bash
    git clone https://github.com/audacity/audacity.git
    cd audacity
    git checkout release-3.7.0
    ```

2. Crie a pasta de build e compile:
    ```bash
    mkdir build
    cd build
    cmake -G "Unix Makefiles" .. -DCMAKE_BUILD_TYPE=Release
    make -j$(nproc)
    sudo make install
    ```

3. Ative o módulo no Audacity:
    - Abra o Audacity e vá em `Editar > Preferências > Módulos`.
    - Encontre `mod-openvino` e ative-o.
    - Reinicie o Audacity.

---

Seguindo esses passos atualizados com base no tutorial oficial da Intel, você terá o Audacity configurado com o plugin OpenVINO no Ubuntu 24.04.
