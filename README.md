
# Instalação do Plugin Intel OpenVINO no Audacity 3.7 no Ubuntu 24.04

## Introdução
Este tutorial descreve como instalar o plugin Intel OpenVINO no Audacity 3.7 em um ambiente Ubuntu 24.04.

---
## Passo 1: Tenho uma GPU?
Sem GPU, não vai dar certo. Por sorte, todas as plataformas atuais, incluindo as Intel Onboard são GPUs também. Então confira qual a sua GPU e instale os drivers adequados.  

# Se tiver uma GPU Intel, execute:
```bash
sudo apt install -y intel-gpu-tools
```
# Se tiver uma GPU AMD:
```bash
sudo apt install mesa-opencl-icd
```
# Se tiver uma GPU nVIDIA, tome cautela, vou assumir o driver 535, porém pode ser outro:
```bash
sudo apt install nvidia-driver-535 (ou driver correspondente)
sudo apt install nvidia-opencl-icd-535 (ou driver correspondente)
```
---
## Passo 2: Tenho OpenCL instalado e funcional? 
Execute o 'clinfo' no terminal, se ele responder:
```bash
Number of platforms                               0

ICD loader properties
  ICD loader Name                                 OpenCL ICD Loader
  ICD loader Vendor                               OCL Icd free software
  ICD loader Version                              2.3.2
  ICD loader Profile                              OpenCL 3.0
```
É porque você não tem uma GPU habilitada em seu sistema e será inutil prosseguir. Você deve resolver este problema antes.
---
## Passo 3: Instale as dependências necessárias

```bash
sudo apt update
sudo apt install -y clinfo
sudo apt install -y build-essential cmake git python3-pip libgtk2.0-dev libasound2-dev libjack-jackd2-dev uuid-dev ocl-icd-opencl-dev
sudo apt install -y python3.12-venv
python3 -m venv ~/.venvs/conan
source ~/.venvs/conan/bin/activate
pip install conan (talvez desnecessário)

## Se tiver uma GPU Intel, execute:
sudo apt install -y intel-gpu-tools
## Se tiver uma GPU nVIDIA:
sudo apt install nvidia-driver-535 (ou driver correspondente)
sudo apt install nvidia-opencl-icd-535 (ou driver correspondente)
## Se tiver uma GPU AMD:
sudo apt install mesa-opencl-icd
```

---


## Passo 4: Baixe e instale o OpenVINO

1. Acesse a [página de downloads do OpenVINO](https://github.com/openvinotoolkit/openvino/releases) e localize o pacote para Ubuntu 24.04:
    ```bash
    mkdir ~/Downloads/openvino
    cd ~/Downloads/openvino
    wget -vc https://storage.openvinotoolkit.org/repositories/openvino/packages/2024.5/linux/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
    ```

2. Extraia o arquivo:
    ```bash
    tar xvf l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
    rm -f l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
    ```

3. Instale as dependências do OpenVINO:
    ```bash
    cd l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/install_dependencies/
    sudo -E ./install_openvino_dependencies.sh
    ```

4. Configure o ambiente:
    ```bash
    source ~/Downloads/openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/setupvars.sh
    ```

5. Para carregar automaticamente as variáveis em novas sessões:
    ```bash
    echo "source ~/Downloads/openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/setupvars.sh" >> ~/.bashrc
    ```

---

## Passo 5: Baixe e instale o Libtorch

1. Baixe a versão C++ do Libtorch:
    ```bash
    wget https://download.pytorch.org/libtorch/cpu/libtorch-cxx11-abi-shared-with-deps-2.4.1%2Bcpu.zip
    ```

2. Extraia o arquivo:
    ```bash
    unzip libtorch-cxx11-abi-shared-with-deps-2.4.1+cpu.zip
    ```

3. Configure a variável de ambiente:
    ```bash
    export LIBTORCH_ROOTDIR=$(pwd)/libtorch
    ```

---

## Passo 5: Compile o `whisper.cpp` com suporte ao OpenVINO

1. Clone o repositório:
    ```bash
    git clone https://github.com/ggerganov/whisper.cpp
    cd whisper.cpp
    git checkout v1.5.4
    ```

2. Crie a pasta de build e compile:
    ```bash
    mkdir build
    cd build
    cmake .. -DWHISPER_OPENVINO=ON
    make -j$(nproc)
    ```

3. Instale o `whisper.cpp`:
    ```bash
    cmake --install . --config Release --prefix ./installed
    ```

4. Configure as variáveis de ambiente:
    ```bash
    export WHISPERCPP_ROOTDIR=$(pwd)/installed
    export LD_LIBRARY_PATH=${WHISPERCPP_ROOTDIR}/lib:$LD_LIBRARY_PATH
    ```

---

## Passo 7: Compile o Audacity com o módulo OpenVINO

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
    ```

3. Clone o repositório do módulo OpenVINO para o Audacity:
    ```bash
    git clone https://github.com/intel/openvino-plugins-ai-audacity.git
    ```

4. Copie o módulo para o diretório de módulos do Audacity:
    ```bash
    cp -r openvino-plugins-ai-audacity/mod-openvino ../modules/
    ```

5. Edite o arquivo `CMakeLists.txt` no diretório `modules` do Audacity:
    ```bash
    nano ../modules/CMakeLists.txt
    ```

    Adicione o seguinte após o comentário:
    ```cmake
    add_subdirectory(mod-openvino)
    ```

6. Recompile o Audacity:
    ```bash
    cd ../build
    cmake -G "Unix Makefiles" .. -DCMAKE_BUILD_TYPE=Release
    make -j$(nproc)
    sudo make install
    ```

---

## Passo 8: Ative o módulo OpenVINO no Audacity

1. Inicie o Audacity:
    ```bash
    audacity
    ```

2. Vá em `Editar > Preferências > Módulos`.

3. Encontre o módulo `mod-openvino` e altere seu estado para **Ativado**.

4. Reinicie o Audacity para que as mudanças tenham efeito.

---

Seguindo esses passos, o plugin Intel OpenVINO estará instalado e funcionando no Audacity 3.7 no Ubuntu 24.04.
