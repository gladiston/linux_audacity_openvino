
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
Entenda que saber qual placa é, e saber que driver está usando, são coisas distintas. 
Precisaremos atualizar o repositório e instalar o 'clinfo':
```bash
sudo apt update
sudo apt install -y clinfo mesa-opencl-icd
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
'**Number of platforms = 0**' é ruim porque significa que você não tem uma GPU habilitada em seu sistema para rodar opencl e será inutil prosseguir. 
Drivers para placas de video intel ou amd são opensources e geralmente são instaladas automaticamente, mas as vezes algumas coisas estão faltando, veja:

**Se tiver uma GPU Intel, talvez precise instalar este programa:**
```bash
sudo apt install -y intel-gpu-tools
```

**Se tiver uma GPU nVIDIA, tome cautela, vou assumir o driver 535 que é o mais atualizado quando criei este HowTo, porém já pode haver outro mais recente ou apropriado:**
```bash
sudo apt install nvidia-driver-535 (ou driver correspondente)
sudo apt install nvidia-opencl-icd-535 (ou driver correspondente)
```
Agora, tente novamente:
```bash
Number of platforms                               0

ICD loader properties
  ICD loader Name                                 OpenCL ICD Loader
  ICD loader Vendor                               OCL Icd free software
  ICD loader Version                              2.3.2
  ICD loader Profile                              OpenCL 3.0
```
'**Number of platforms = 0**' significa que não obtivêmos êxito, tente descobrir o problema usando lista de discussão, foruns e o chatgpt, mas não prossiga com este tutorial sem resolver este problema antes.

## Passo 3: Instale as dependências necessárias

```bash
sudo apt install -y build-essential cmake git python3-pip libgtk2.0-dev libasound2-dev libjack-jackd2-dev uuid-dev ocl-icd-opencl-dev
sudo apt install -y python3.12-venv
python3 -m venv ~/.venvs/conan
source ~/.venvs/conan/bin/activate
```
Execute também:
```bash
source ~/.venvs/conan/bin/activate
```
isso fará abrir uma especie shell do python, daí você executa:
```bash
(conan) ~➤ pip install conan
```
O comando 'pip install conan' irá instalar o 'conan' neste ambiente do python.

## Passo 4: Baixe e instale o OpenVINO

1. Vá até uma pasta onde deseje que a pasta a ser 'baixada' dê origem a subpasta do openvino, ex:
    ```bash
    cd /usr/local    
    ```
    
2. Acesse a [página de downloads do OpenVINO](https://github.com/openvinotoolkit/openvino/releases) e localize o pacote para Ubuntu 24.04 e faça o download para a pasta '~/Downloads/openvino'. Se tiver usando o Ubuntu 24.04, poderá baixá-lo pelo terminal:
    ```bash
    sudo wget -vc https://storage.openvinotoolkit.org/repositories/openvino/packages/2024.5/linux/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
    ```
3. Extraia o arquivo:
    ```bash
    sudo tar xvf l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
    sudo rm -f l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
    ```
 
4. Renomeamos a pasta extraida para 'openvino'
   ```bash
   sudo mv l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64 openvino
   ```
5 Dê permssões globais a esta pasta:
    ```bash
    sudo chmod 755 -vR /usr/local/openvino
    sudo chown -vR $USER /usr/local/openvino
    ```  
   
6. Guarde o caminho dessa pasta, ela é a nossa pasta raiz, a saber:
   ```bash
   /usr/local/openvino
   ```
   Ela será muito referenciada no restante do artigo.
7. Instale as dependências do OpenVINO:
    ```bash
    cd /usr/local/openvino/install_dependencies/
    sudo -E ./install_openvino_dependencies.sh
    ```
    
8. Configure o ambiente:
    ```bash
    source /usr/local/openvino/setupvars.sh
    ```

9. Para carregar automaticamente as variáveis em novas sessões(opcional, use apenas se precisa lidar mais vezes com ele):
    ```bash
    echo "source /usr/local/openvino/setupvars.sh" >> ~/.bashrc
    ```
## Passo 5: OpenVINO Tokenizers Extension 
1. Vá para a pasta de referencia que esta sempre dentro da 'raiz' e :
    ```bash
    cd /usr/local/openvino/install_dependencies
    ```
2. Use o navegador e vá até a página https://storage.openvinotoolkit.org/repositories/openvino_tokenizers/packages/2024.5.0.0/[https://storage.openvinotoolkit.org/repositories/openvino_tokenizers/packages/2024.5.0.0/] e baixe a versão correspondente ao seu Ubuntu, exemplo:
  ```bash
  cd /usr/local/openvino/install_dependencies/
  sudo wget -vc https://storage.openvinotoolkit.org/repositories/openvino_tokenizers/packages/2024.5.0.0/openvino_tokenizers_ubuntu24_2024.5.0.0_x86_64.tar.gz
  ```
  Extraia os arquivos:
  ```bash
  sudo tar zxvf openvino_tokenizers_ubuntu24_2024.5.0.0_x86_64.tar.gz
  sudo rm -f openvino_tokenizers_ubuntu24_2024.5.0.0_x86_64.tar.gz
  ```
Serão extraidos os seguintes arquivos na pasta runtime/lib/intel64/ e que nos interessam:
  ```
  (...)
  runtime/lib/intel64/libcore_tokenizers.so
  runtime/lib/intel64/libopenvino_tokenizers.so
  ```
Precisartemos copiá-los para: (cuidado, linha muito extensa)
  ```bash
  sudo cp -r /usr/local/openvino/install_dependencies/runtime/lib/intel64/* \
             /usr/local/openvino/runtime/lib/intel64
  ```
(Recomendação) Crie e ative 'virtual env':
```bash
sudo python3 -m venv venv
source venv/bin/activate
```

## Passo 5: Baixe e instale o Libtorch

1. Vá para a pasta de referencia que esta sempre dentro da 'raiz' e :
    ```bash
    cd /usr/local/openvino/install_dependencies
    ```
    
2. Baixe a versão C++ do Libtorch:
    ```bash
    cd /usr/local/openvino/install_dependencies
    sudo wget https://download.pytorch.org/libtorch/cpu/libtorch-cxx11-abi-shared-with-deps-2.4.1%2Bcpu.zip
    ```

3. Extraia o arquivo:
    ```bash
    sudo unzip libtorch-cxx11-abi-shared-with-deps-2.4.1+cpu.zip
    sudo rm -f libtorch-cxx11-abi-shared-with-deps-2.4.1+cpu.zip
    ```

4. Configure a variável de ambiente:
    ```bash
    export LIBTORCH_ROOTDIR=/usr/local/openvino/install_dependencies/libtorch
    ```
   Vamos acrenscentar os caminhos acima ao nosso terminal, caso venhamos a usar de novo:
   ```bash
   nano ~/.bashrc
   ```
   E acrescente a ultima ou penultimas linhas, como preferir:
   ```bash
   export LIBTORCH_ROOTDIR=/usr/local/openvino/install_dependencies/libtorch
   ```

## Passo 6: Compile o `whisper.cpp` com suporte ao OpenVINO
1. Vá para a pasta de referencia que esta sempre dentro da 'raiz' e :
    ```bash
    cd /usr/local/openvino/install_dependencies
    ```
    
2. Verifique quantos nucleos/threads serão suportados pelo compilador:
   ```bash
   echo $(nproc)
   ```
   A resposta será 2, 4, 6, 8, 10, 12... que será a quantidade de nucleos/threads que seu computador suportará durante a compilação usando 'make -j', quanto mais, melhor. Se a resposta for vazia, **APENAS SE FOR VAZIA**, você terá que especificar na mão, eu usarei 4 no exemplo abaixo, mas se seu computador suportar mais, especifique o quanto usará:
```bash
   set nproc=4
   ```
 
3. Clone o repositório:
    ```bash
    cd /usr/local/openvino
    source /usr/local/openvino/setupvars.sh
    sudo git clone https://github.com/ggerganov/whisper.cpp
    sudo chown -vR $USER /usr/local/openvino
    sudo git config --global --add safe.directory  /usr/local/openvino/whisper.cpp
    cd whisper.cpp
    sudo git checkout v1.5.4
    sud bash ./models/download-ggml-model.sh base.en
    cd ..
    ```

4. Crie a pasta de build e compile:
    ```bash
    sudo mkdir build
    cd build
    sudo cmake ../whisper.cpp/ -DWHISPER_OPENVINO=ON
    sudo make -j$(nproc)
    sudo make install
    ```
    Os arquivos do 'whisper' que nos importam foram instalados em:
    ```
    /usr/local/lib/libwhisper.so
    /usr/local/include/ggml.h
    /usr/local/include/whisper.h
    ```
    
5. Configure as variáveis de ambiente:
    ```bash
    export WHISPERCPP_ROOTDIR=/usr/local/lib
    export LD_LIBRARY_PATH=${WHISPERCPP_ROOTDIR}/lib:$LD_LIBRARY_PATH
    sudo ldconfig
    ```
   Vamos acrenscentar os caminhos acima ao nosso terminal, caso venhamos a usar de novo:
   ```bash
   nano ~/.bashrc
   ```
   E acrescente na ultima ou penultimas linhas, como preferir:
   ```bash
   export WHISPERCPP_ROOTDIR=/usr/local/lib
   export LD_LIBRARY_PATH=${WHISPERCPP_ROOTDIR}/lib:$LD_LIBRARY_PATH
   ```
## Passo 7: Baixe os fontes do Audacity:
1. Vá para a pasta onde teremos o audacity baixado  :
    ```bash
    cd /usr/local
    ```
    
2. Clone o repositório do Audacity e se posione na pasta 'audacity':
    ```bash
    sudo git clone https://github.com/audacity/audacity.git
    sudo chmod -vR 755 /usr/local/audacity
    sudo chown -vR $USER /usr/local/audacity
    sudo git config --global --add safe.directory /usr/local/audacity
    cd audacity
    sudo git reset --hard   
    sudo git checkout release-3.7.0    
    ```
    Por enquanto, vamos apenas baixá-los, mas num dos passos posteriores também iremos compilá-lo.
   
## Passo 8: Baixar e compilar o módulo-plugin OpenVINO
1. Vá para a pasta onde vamos baixar 'openvino-plugins-ai-audacity':
    ```bash
    cd /usr/local/openvino
    ``` 
2. Clone o repositório do módulo-plugin OpenVINO dentro da pasta do Audacity:
    ```bash
    sudo git clone https://github.com/intel/openvino-plugins-ai-audacity.git
    sudo chmod -vR 755 /usr/local/openvino
    sudo chown -vR $USER /usr/local/openvino
    sudo git config --global --add safe.directory /usr/local/openvino/openvino-plugins-ai-audacity
    sudo git reset --hard
    sudo git checkout release-3.7.0    
    ```   
3. Copie o módulo para o diretório de módulos do Audacity:
    ```bash
    sudo cp -r ./openvino-plugins-ai-audacity/mod-openvino \
               /usr/local/audacity/modules/    
    ```
4. Edite o arquivo `CMakeLists.txt` no diretório `modules` do Audacity:
    ```bash
    sudo nano /usr/local/audacity/modules/CMakeLists.txt
    ```

    Adicione o seguinte linha abaixo do comentário "# Include the modules that we'll build":
    ```bash
    add_subdirectory(mod-openvino)
    ```
    O que você fez foi colocar no radar da compilação do Audacity, o modulo openvino.
   
## Passo 9: Compile o Audacity com o módulo OpenVINO
1. Vá para a pasta onde baixamos o 'Audacity':
    ```bash
    cd /usr/local/audacity
    ``` 
    
2. Vamos preparar o ambiente para compilação:
    ```bash
    source /usr/local/openvino/setupvars.sh
    ```
    Vamos exportar as variaveis de ambiente novamente caso não esteja continuando desde o principio do artigo:
    ```bash   
    export LIBTORCH_ROOTDIR=/usr/local/openvino/install_dependencies/libtorch
    export WHISPERCPP_ROOTDIR=/usr/local/lib
    export LD_LIBRARY_PATH=${WHISPERCPP_ROOTDIR}/lib:$LD_LIBRARY_PATH
    sudo ldconfig
    ```      
    Vamos criar uma pasta de build:
    ```bash
    mkdir audacity-build
    cd audacity-build
    ```
    Vamos compilar:
    ```bash    
    sudo cmake -G "Unix Makefiles" .. -DCMAKE_BUILD_TYPE=Release
    sudo make -j$(nproc)
    ```
    Vamos instalar:
    ```bash   
    sudo make install
    ```
    O comando **sudo make install** colocará os binários do Audacity no radar de seu sistema operacional, por essa razão, você não deve ter o audacity previamente instalado, pois o mesmo seria sobreposto. 
 
## Último passo: Ative o módulo **mod-openvino** no aplicativo Audacity

1. Inicie o Audacity, neste momento ele já estará instalado em /usr/local/bin e aparecerá no menu do seu ambiente de desktop, mas por alguma razão que ainda não consegui identificar, ele só pode ser carregado dessa forma:
    ```bash
    /usr/local/audacity/audacity-build/Release/bin/audacity
    ```
    Caso contrário uma mensagem de erro indicando 'Incapaz de carregar o modulo mod-openvino': Erro: ioctl inapropriado para dispositivo', o qual ainda não descobri, o porquê, tenho suspeitas de que é porque as variaveis de ambientes criadas para executar no terminal não são carregadas numa execução direta do ambiente gráfico, mas carece de mais pesquisas para confirmar.
    Um 'workaround' é usar o programa de editor de menus chamado 'menulibre' e consertar a chamada do Audacity para o diretorio acima:
    ```bash
    sudo apt install menulibre
    ```
    Depois escolha no menu do seu sistema, menulibre e procure pelo 'Audacity', geralmente ele fica na seção 'Multimedia' e troque:
    **env GDK_BACKEND=x11 UBUNTU_MENUPROXY=0 audacity %F**
    por
    **/usr/local/audacity/audacity-build/Release/bin/audacity %F**
   Também é imprescindivel marcar a opção **Executar pelo terminal** senão, não funcionará.
    
   
3. Dentro do Audacity vá em `Editar > Preferências > Módulos`.

4. Encontre o módulo `mod-openvino` e altere seu estado para **Ativado**.

5. Reinicie o Audacity para que as mudanças tenham efeito.

---

Seguindo esses passos, o plugin Intel OpenVINO estará instalado e funcionando no Audacity 3.7 no Ubuntu 24.04.
Este script foi adaptado originalmente das instruçoes em: https://github.com/intel/openvino-plugins-ai-audacity/blob/main/doc/build_doc/linux/README.md
