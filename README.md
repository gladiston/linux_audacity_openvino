
# Instalação do Plugin Intel OpenVINO no Audacity 3.7 no Ubuntu 24.04

## Introdução
Este tutorial descreve como instalar o plugin Intel OpenVINO no Audacity 3.7 em um ambiente Ubuntu 24.04. 
Com o Audacity e o plugin Intel OpenVINO, você pode potencializar tarefas de edição e processamento de áudio com inteligência artificial, como transcrição de fala para texto, supressão de ruídos, separação de fontes (vocal/instrumentos), reconhecimento de emoções e idiomas, tradução automática, e aplicação de efeitos estilizados. Além disso, o OpenVINO permite processamento em tempo real, diagnóstico acústico avançado e otimização de desempenho em hardware Intel, tornando o Audacity uma ferramenta ainda mais poderosa para produção de áudio profissional ou amadora.

Para prosseguir tenha certeza de não ter uma versão do Audacity instalada, pois as distribuições ainda não estão usando a versão 3.7. E as que vem de repositórios snap ou flathub não foram compiladas para usar o módulo do OpenVINO. 
O que esse tutorial pretende fazer é compilar o Audacity junto com o modulo OpenVINO a partir dos codigos fontes, então haverá muitas dependencias a serem instaladas que se você nunca compilou ou nunca compilará mais nada é um pouco de disperdicio ter toneladas de programas apenas para ter o Audacity+OpenVINO instalado em seu computador.

---
## Passo 1: Tenho uma GPU?
Sem GPU, não vai dar certo. Por sorte, todas as plataformas atuais, incluindo as Intel Onboard são GPUs também. Então confira qual a sua GPU e instale os drivers adequados. Você pode descobrir que placa de vídeo possui simplesmente executando no terminal:
```bash
sudo lshw -C display
``` 
Ententa que saber qual placa é, e saber que driver está usando, são coisas distintas.

**Se tiver uma GPU Intel, execute:**
```bash
sudo apt install -y intel-gpu-tools
```

**Se tiver uma GPU AMD:**
```bash
sudo apt install mesa-opencl-icd
```

**Se tiver uma GPU nVIDIA, tome cautela, vou assumir o driver 535, porém pode ser outro:**
```bash
sudo apt install nvidia-driver-535 (ou driver correspondente)
sudo apt install nvidia-opencl-icd-535 (ou driver correspondente)
```
## Passo 2: Tenho OpenCL instalado e funcional? 
Execute no terminal:
```bash
sudo ldconfig
clinfo
```
Se a saída do comando for algo como:
```bash
Number of platforms                               0

ICD loader properties
  ICD loader Name                                 OpenCL ICD Loader
  ICD loader Vendor                               OCL Icd free software
  ICD loader Version                              2.3.2
  ICD loader Profile                              OpenCL 3.0
```
'**Number of platforms = 0**' é ruim porque significa que você não tem uma GPU habilitada em seu sistema para rodar opencl e será inutil prosseguir. Você deve resolver este problema antes.

## Passo 3: Instale as dependências necessárias

```bash
sudo apt update
sudo apt install -y clinfo
sudo apt install -y build-essential cmake git python3-pip libgtk2.0-dev libasound2-dev libjack-jackd2-dev uuid-dev ocl-icd-opencl-dev
sudo apt install -y python3.12-venv
python3 -m venv ~/.venvs/conan
source ~/.venvs/conan/bin/activate
```
~~pip install conan~~ (talvez desnecessário)

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
## Passo 5: OpenVINO Tokenizers Extension 
Vá até a página https://storage.openvinotoolkit.org/repositories/openvino_tokenizers/packages/2024.5.0.0/[https://storage.openvinotoolkit.org/repositories/openvino_tokenizers/packages/2024.5.0.0/] e baixe a versão correspondente ao seu Ubuntu, exemplo:
  ```bash
  wget -vc https://storage.openvinotoolkit.org/repositories/openvino_tokenizers/packages/2024.5.0.0/openvino_tokenizers_ubuntu24_2024.5.0.0_x86_64.tar.gz
  tar zxvf openvino_tokenizers_ubuntu24_2024.5.0.0_x86_64.tar.gz
  rm -f openvino_tokenizers_ubuntu24_2024.5.0.0_x86_64.tar.gz
  ```
Serão extraidos os seguintes arquivos na pasta runtime/lib/intel64/ e que nos interessam:
  ```
  (...)
  runtime/lib/intel64/libcore_tokenizers.so
  runtime/lib/intel64/libopenvino_tokenizers.so
  ```
Precisartemos copiá-los para: (cuidado, linha muito extensa)
  ```bash
  cp -r ~/Downloads/openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/install_dependencies/runtime/lib/intel64/* ~/Downloads/openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/runtime/lib/intel64
  ```

## Passo 5: Baixe e instale o Libtorch

1. Baixe a versão C++ do Libtorch:
    ```bash
    wget https://download.pytorch.org/libtorch/cpu/libtorch-cxx11-abi-shared-with-deps-2.4.1%2Bcpu.zip
    ```

2. Extraia o arquivo:
    ```bash
    unzip libtorch-cxx11-abi-shared-with-deps-2.4.1+cpu.zip
    rm -f libtorch-cxx11-abi-shared-with-deps-2.4.1+cpu.zip
    ```

3. Configure a variável de ambiente:
    ```bash
    export LIBTORCH_ROOTDIR=$(pwd)/libtorch
    ```
## Passo 6: Compile o `whisper.cpp` com suporte ao OpenVINO

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
    cmake .. -DWHISPER_OPENVINO=ON -DCMAKE_INSTALL_PREFIX=/usr/local
    make -j$(nproc)
    sudo make install
    ```
    
4. Configure as variáveis de ambiente:
    ```bash
    export WHISPERCPP_ROOTDIR=/usr/local/lib
    export LD_LIBRARY_PATH=${WHISPERCPP_ROOTDIR}/lib:$LD_LIBRARY_PATH
    sudo ldconfig
    ```

## Passo 7: Compile o Audacity com o módulo OpenVINO

1. Clone o repositório do Audacity:
    ```bash
    cd ~/Downloads/openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64
    git clone https://github.com/audacity/audacity.git
    cd audacity
    git checkout release-3.7.0
    ```

2. Clone o repositório do módulo plugin OpenVINO dentro da pasta do Audacity:
    ```bash
    git clone https://github.com/intel/openvino-plugins-ai-audacity.git
    ```   
3. Copie o módulo para o diretório de módulos do Audacity:
    ```bash
    cp -r openvino-plugins-ai-audacity/mod-openvino ./modules/    
    ```
4. Edite o arquivo `CMakeLists.txt` no diretório `modules` do Audacity:
    ```bash
    nano ./modules/CMakeLists.txt
    ```

    Adicione o seguinte linha:
    ```bash
    add_subdirectory(mod-openvino)
    ```
    
   Ficando assim no final:
   
   ```bash
    foreach( MODULE ${MODULES} )
       add_subdirectory("${MODULE}")
    endforeach()

    *add_subdirectory(mod-openvino)*

    if( NOT CMAKE_SYSTEM_NAME MATCHES "Darwin" )
       if( NOT "${CMAKE_GENERATOR}" MATCHES "Visual Studio*")
          install( DIRECTORY "${_DEST}/modules"
                   DESTINATION "${_PKGLIB}" )
       endif()
    endif()
   ```
 
5. Crie a pasta de build e compile o Audacity:
    ```bash
    cd ~/Downloads/openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/audacity
    mkdir build
    cd build
    cmake -G "Unix Makefiles" .. -DCMAKE_BUILD_TYPE=Release
    make -j$(nproc)
    sudo make install
    ```

## Último passo: Ative o módulo OpenVINO no Audacity

1. Inicie o Audacity:
    ```bash
    audacity
    ```

2. Vá em `Editar > Preferências > Módulos`.

3. Encontre o módulo `mod-openvino` e altere seu estado para **Ativado**.

4. Reinicie o Audacity para que as mudanças tenham efeito.

---

Seguindo esses passos, o plugin Intel OpenVINO estará instalado e funcionando no Audacity 3.7 no Ubuntu 24.04.
Este script foi adaptado originalmente das instruçoes em: https://github.com/intel/openvino-plugins-ai-audacity/blob/main/doc/build_doc/linux/README.md
