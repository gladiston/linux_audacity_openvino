Construção do Módulo OpenVINO para o Audacity no Linux (Ubuntu 24.04)
---
Este tutorial descreve como instalar o plugin Intel OpenVINO no Audacity 3.7 em um ambiente Ubuntu 24.04. Com o Audacity e o plugin Intel OpenVINO, você pode potencializar tarefas de edição e processamento de áudio com inteligência artificial, como transcrição de fala para texto, supressão de ruídos, separação de fontes (vocal/instrumentos), reconhecimento de emoções e idiomas, tradução automática, e aplicação de efeitos estilizados. Além disso, o OpenVINO permite processamento em tempo real, diagnóstico acústico avançado e otimização de desempenho em hardware Intel, tornando o Audacity uma ferramenta ainda mais poderosa para produção de áudio profissional ou amadora.

Para prosseguir tenha certeza de não ter uma versão do Audacity instalada, pois as distribuições ainda não estão usando a versão 3.7. E as que vem de repositórios snap ou flathub não foram compiladas para usar o módulo do OpenVINO. O que esse tutorial pretende fazer é compilar o Audacity junto com o modulo OpenVINO a partir dos codigos fontes, então haverá muitas dependencias a serem instaladas que se você nunca compilou ou nunca compilará mais nada é um pouco de disperdicio ter toneladas de programas apenas para ter o Audacity+OpenVINO instalado em seu computador.

Visão Geral
---
Antes de entrar nos detalhes, aqui está uma visão geral das etapas:

1. Clonar e construir o `whisper.cpp` com suporte ao OpenVINO (para o módulo de transcrição do Audacity).
2. Clonar e construir o Audacity sem modificações (para garantir que a construção "pura" funcione corretamente).
3. Adicionar nosso módulo OpenVINO ao código-fonte do Audacity e reconstruí-lo.

**Nota:** Ao longo da documentação, utilizamos `~/audacity-openvino/` como diretório de trabalho padrão, onde os pacotes e componentes serão baixados e o projeto será construído. Sinta-se à vontade para usar essa estrutura ou configurar um diretório em outro local do seu sistema com nomes que funcionem melhor para você. Apenas certifique-se de ajustar os comandos conforme necessário.

Passo 1: Tenho o clinfo e o mesa-opencl-icd?
---
1. Precisaremos atualizar o repositório e instalar o 'clinfo' e 'mesa-opencl-icd':
```bash
sudo apt update
sudo apt install -y clinfo mesa-opencl-icd
```

Passo 2: Tenho OpenCL instalado e funcional? 
---
Execute no terminal:
```bash
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
sudo apt install -y nvidia-driver-535 (ou driver correspondente)
sudo apt install -y nvidia-opencl-icd-535 (ou driver correspondente)
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


Passo 3: Instale as dependências necessárias
---

```bash
sudo apt install -y build-essential cmake git python3-pip libgtk2.0-dev libasound2-dev libjack-jackd2-dev uuid-dev ocl-icd-opencl-dev
sudo apt install -y tree python3.12-venv
```

Passo 4: Vamos criar o dretorio base
---
```bash
mkdir ~/audacity-openvino
cd ~/audacity-openvino
```

Passo 5: OpenVINO
---
1. Acesse a [página de downloads do OpenVINO](https://github.com/openvinotoolkit/openvino/releases) e localize o pacote para Ubuntu 24.04 e faça o download para a pasta HOME. Nesta instalação usaremos o arquivo apropriado para o Ubuntu 24.04:
```bash
wget -vc https://storage.openvinotoolkit.org/repositories/openvino/packages/2024.5/linux/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
```
2. Extraia o arquivo:
```bash
tar xvf l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
```
3. Renomeamos a pasta extraida para 'openvino_toolkit'
```bash
mv l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64 openvino_toolkit
```
4. Remova o arquivo baixado:
```bash
rm -f l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
```
A pasta '~/audacity-openvino' será muito referenciada no restante do artigo por causa das subpastas contidas nele.


Passo 6: Instale as dependências do OpenVINO:
---
```bash
cd ~/audacity-openvino/openvino_toolkit/install_dependencies
sudo -E ./install_openvino_dependencies.sh
cd ..
# Configurar o ambiente
source setupvars.sh
```


Passo 7: Extensão OpenVINO Tokenizers
---
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino/openvino_toolkit/install_dependencies
```
2. Use o navegador e vá até a página https://storage.openvinotoolkit.org/repositories/openvino_tokenizers/packages/2024.5.0.0/[https://storage.openvinotoolkit.org/repositories/openvino_tokenizers/packages/2024.5.0.0/] e baixe a versão correspondente ao seu Ubuntu, exemplo:
```bash
wget -vc https://storage.openvinotoolkit.org/repositories/openvino_tokenizers/packages/2024.5.0.0/openvino_tokenizers_ubuntu24_2024.5.0.0_x86_64.tar.gz
```
3. Extraia os arquivos:
```bash
tar zxvf openvino_tokenizers_ubuntu24_2024.5.0.0_x86_64.tar.gz
```
Serão extraidos os seguintes arquivos na pasta runtime/lib/intel64/ e que nos interessam:
```
(...)
runtime/lib/intel64/libcore_tokenizers.so
runtime/lib/intel64/libopenvino_tokenizers.so
```
4. Precisartemos copiá-los para: (cuidado, linha muito extensa)
```bash
cp -r ~/audacity-openvino/openvino_toolkit/install_dependencies/runtime/lib/intel64/* \
      ~/audacity-openvino/openvino_toolkit/runtime/lib/intel64
```
5. Remova o arquivo baixado:
```bash
rm -f openvino_tokenizers_ubuntu24_2024.5.0.0_x86_64.tar.gz
```


Passo 8: Libtorch (distribuição C++ do PyTorch)
---
  Dependência para muitos dos pipelines portados do PyTorch (musicgen, htdemucs, etc). Estamos usando a versão `libtorch-cxx11-abi-shared-with-deps-2.4.1+cpu.zip`.
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```
2. Baixe a versão C++ do Libtorch:
```bash
wget https://download.pytorch.org/libtorch/cpu/libtorch-cxx11-abi-shared-with-deps-2.4.1%2Bcpu.zip
```
3. Extraia o arquivo:
```bash
unzip libtorch-cxx11-abi-shared-with-deps-2.4.1+cpu.zip
```
4. Para configurar o ambiente
```bash
export LIBTORCH_ROOTDIR=$HOME/audacity-openvino/libtorch
```
5. Remova o arquivo baixado:
```bash
rm -f libtorch-cxx11-abi-shared-with-deps-2.4.1+cpu.zip
```

Passo 9: OpenCL
---
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```
2. Para otimizar o desempenho em GPUs, utilizamos o OpenCL com APIs de interoperabilidade para o OpenVINO. Portanto, precisamos instalar os pacotes de desenvolvimento do OpenCL:
```bash
sudo apt install -y ocl-icd-opencl-dev
```
3. Verifique quantos nucleos/threads serão suportados pelo compilador:
```bash
echo $(nproc)
```
A resposta será 2, 4, 6, 8, 10, 12... que será a quantidade de nucleos/threads que seu computador suportará durante a compilação usando 'make -j', quanto mais, melhor. Se a resposta for vazia, **APENAS SE FOR VAZIA**, você terá que especificar na mão, eu usarei 4 no exemplo abaixo, mas se seu computador suportar mais, especifique o quanto usará:
```bash
set nproc=4
```

Passo 10: Construção de Subcomponentes
---
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```
2. Vamos baixar o arquivo em:
```bash
wget -vc https://storage.openvinotoolkit.org/repositories/openvino/packages/2024.5/linux/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
```
3. Descompactamos o arquivo:
```bash
tar zxvf l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
```
4. Preparamos o ambiente:
```bash
source $HOME/audacity-openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/setupvars.sh
```
O Audacity+OpenVINO só funcionará com o ambiente preparado, não se esqueça disso quando tentar executá-lo e não funcionr. Por isso, se usa muito o Audacity e fará uso do OpenVINO é recomendável colocar a linha acima dentro do ~/.bashrc, execute:
```bash
echo source $HOME/audacity-openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/setupvars.sh>>~/.bashrc
```

5. Isso é necessário para os procedimentos a seguir, depois removemos o arquivo baixado:
```bash
rm -f l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
```


Passo 11: Construção de Subcomponentes
---
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```
2. Agora vamos construir o `whisper.cpp`. Primeiro vamos baixá-lo:
```bash
git clone https://github.com/ggerganov/whisper.cpp
```
3. Depois estabelecer a versão que desejamos do whisper.cpp:
```bash
cd whisper.cpp
git checkout v1.5.4
```
4. E agora, vamos criar pasta de construção e compilá-lo:
```bash
cd ..
mkdir whisper-build
cd whisper-build
```
5. Vamos agora rodar o cmake, especificadno que desejamos habiitar o suporte ao OpenVINO:
```bash
cmake ../whisper.cpp/ -DWHISPER_OPENVINO=ON
```
6. Agora fazemos o processo de build:
```bash
make -j$(nproc)
```
7. Ao invés do 'sudo make install' e fazer uma instalação global para o sistema, vamos gerar os binários em './installed':
```bash
cmake --install . --config Release --prefix ./installed
```
8. Notará que em './installed', os arquivos do 'whisper' que nos importam são:
```
/installed/lib/libwhisper.so
/installed/include/ggml.h
/installed/include/whisper.h
```
9. Se você optar por um 'sudo make install' estes arquivos estariam em:
```
/usr/local/lib/libwhisper.so
/usr/local/include/ggml.h
/usr/local/include/whisper.h
```
10. Com a compilação/instalação concluída, a compilação do Audacity encontrará um erro colateral compilado por meio do WHISPERCPP_ROOTDIR. Então você pode configurá-lo assim:
```bash
export WHISPERCPP_ROOTDIR=$HOME/audacity-openvino/whisper-build/installed
export LD_LIBRARY_PATH=${WHISPERCPP_ROOTDIR}/lib:$LD_LIBRARY_PATH
```
(Mas eu vou te lembrar disso mais tarde)

Passo 12: Audacity
---
Ok, vamos prosseguir para a construção real do Audacity. Só um lembrete, primeiro vamos construir o Audacity sem nenhuma modificação. Uma vez feito isso, copiaremos nosso openvino-module para a árvore src do Audacity e o construiremos.
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```
2. Vamos instalar as dependencias para poder compiar o Audacity:
```bash
sudo apt install -y build-essential cmake git python3-pip
sudo apt install -y libgtk2.0-dev libasound2-dev libjack-jackd2-dev uuid-dev
sudo pip3 install conan
```
3. Vamos clonar o repositorio do Audacity:
```bash
git clone https://github.com/audacity/audacity.git
```
4. Vamos informar a versão desejada que iremos compilar:
```bash
cd audacity
git checkout release-3.7.0
```
5. Vamos agora fazer o build fora da pasta do audacity:
```bash
cd ~/audacity-openvino
mkdir audacity-build
cd audacity-build
```
6. Vamos compilá-lo(vá tomar um café, porque será um pouco demorado):
```bash
cmake -G "Unix Makefiles" ../audacity -DCMAKE_BUILD_TYPE=Release
```
7. Depois vamos a uma construção:
```bash
make -j$(nproc)
```
8. Quando isso estiver feito, você pode executar o Audacity assim (do diretório audacity-build):
```bash
./Release/bin/audacity
# ou
$HOME/audacity-openvino/audacity-build/Release/bin/audacity
```
Mas ele ainda ainda não está totalmente pronto até que compilemos o módulo OpenVINO.

Passo 13: Construção do módulo Audacity OpenVINO
---
Agora, vamos percorrer as etapas para realmente construir o módulo Audacity baseado em OpenVINO.
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```
2. Primeiro, clone o seguinte repositório. É aqui que o código real do módulo Audacity vive hoje.
```bash
git clone https://github.com/intel/openvino-plugins-ai-audacity.git
```
3. Precisamos copiar a pasta mod-openvino para a árvore de código-fonte do Audacity. Ou seja, copie a pasta openvino-plugins-ai-audacity/mod-openvino para audacity/modules:
```bash
cp -r openvino-plugins-ai-audacity/mod-openvino/ audacity/modules/
```
4. Agora precisamos editar audacity\modules\CMakeLists.txt para adicionar mod-openvino como um alvo de build. Você só precisa adicionar um **add_subdirectory(mod-openvino)** em algum lugar do arquivo. Por exemplo:
```bash
nano audacity/modules/CMakeLists.txt
```
Ficando assim:
```
# Include the modules that we'll build
add_subdirectory(mod-openvino)

# The list of module sub-folders is ordered so that each folder occurs after any
# others that it depends on
set( FOLDERS
   etc
   import-export
   track-ui
   scripting
   nyquist
   sharing
)

foreach( FOLDER ${FOLDERS} )
   add_subdirectory("${FOLDER}")
endforeach()
```
5. Ok, agora vamos (finalmente) construir o módulo. Aqui está uma recapitulação das variáveis ​​de ambiente que você deveria ter definido:
```bash
# OpenVINO
source $HOME/audacity-openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/setupvars.sh
```
6. Whisper.cpp
```bash
export LIBTORCH_ROOTDIR=$HOME/audacity-openvino/libtorch
```
7. Estamos prontos para recompilar novamente o Audacity
```bash
cd ~/audacity-openvino/audacity-build
```
8. Vamos rodar o processo de cnake novamente
```bash
cmake -G "Unix Makefiles" ../audacity -DCMAKE_BUILD_TYPE=Release
```
9. Novamente o make...
```bash
make -j$(nproc)
```
Se tudo for compilado corretamente, você verá mod-openvino.so em ./Release/lib/audacity/modules/

Passo 14: Executando o audacity pela primeira vez
---
Você já pode prosseguir e executar o audacity, mas somente desse diretorio:
```bash
$HOME/audacity-openvino/audacity-build/Release/bin/audacity
```
Assim que o Audacity estiver aberto, você precisa ir em Edit->Preferences. E no lado esquerdo você verá uma aba 'Modules', clique nela. E aqui você (espero) verá a entrada mod-openvino definida como 'New'. Você precisa alterá-la para 'Enabled'.
Depois disso, feche o audacity e execute-o novamente, porém sempre pelo terminal, assim:
```bash
$HOME/audacity-openvino/audacity-build/Release/bin/audacity
```

Caso contrário uma mensagem de erro indicando 'Incapaz de carregar o modulo mod-openvino': Erro: ioctl inapropriado para dispositivo', poderá aparecer. Eu suspeito que executando de outra forma não funcione porque as variaveis de ambientes criadas para executar pelo terminal não existem quando tenta-se carregá-lo pelo ambiente gráfico, mas carece de mais pesquisas para confirmar.

Passo 15: Instalação de Modelos OpenVINO
---
Para realmente usar esses módulos, precisamos gerar/preencher o diretório `/usr/local/lib/` com os modelos OpenVINO que os plugins irão buscar. Durante a execução, os plugins procurarão esses modelos em um diretório chamado `openvino-models`. Aqui estão os comandos que você pode usar para criar esse diretório e preenchê-lo com os modelos necessários.

⚠️ **Atenção**: Os modelos que esses comandos irão baixar são muito grandes (vários GBs). Tenha cuidado se você estiver usando uma conexão com limite de dados.  

💡 **Sugestão**: Independentemente de estar em uma conexão limitada, se você tiver um dispositivo de armazenamento extra (pen drive ou SSD em um gabinete com 64 GB ou mais), talvez seja interessante salvar esses arquivos de modelo, caso deseje construir tudo isso em outro lugar no futuro. 
## MusicGen
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```
2. Criar um diretório vazio openvino-models para começar
```bash
mkdir openvino-models
```
Criar uma pasta para o MusicGen:
4. Instalar o Git LFS (Large File Storage), necessário para repositórios do Hugging Face
```bash
sudo apt install -y git-lfs
```
5. Criar uma pasta para o MusicGen:
```bash
mkdir openvino-models/musicgen
```
6. Clonar o repositório do Hugging Face:
```bash
git clone https://huggingface.co/Intel/musicgen-static-openvino
```
7. Descompactar o conjunto base de modelos na pasta do MusicGen:
```bash
unzip musicgen-static-openvino/musicgen_small_enc_dec_tok_openvino_models.zip -d ~/audacity-openvino/openvino-models/musicgen
```
8. Descompactar os modelos específicos para mono:
```bash
unzip musicgen-static-openvino/musicgen_small_mono_openvino_models.zip -d ~/audacity-openvino/openvino-models/musicgen
```
9. Descompactar os modelos específicos para estéreo:
```bash
unzip musicgen-static-openvino/musicgen_small_stereo_openvino_models.zip -d ~/audacity-openvino/openvino-models/musicgen
```
10. Excluir o repositório clonado após a extração:
```bash
rm -rf musicgen-static-openvino
```

## Whisper Transcription
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```
2. Clonar o repositório do Hugging Face:
```bash
git clone https://huggingface.co/Intel/whisper.cpp-openvino-models
```
3. Extrair pacotes de modelos individuais para o diretório `openvino-models`:
```bash
unzip whisper.cpp-openvino-models/ggml-base-models.zip -d ~/audacity-openvino/openvino-models
```
```bash
unzip whisper.cpp-openvino-models/ggml-small-models.zip -d ~/audacity-openvino/openvino-models
```
```bash
unzip whisper.cpp-openvino-models/ggml-small.en-tdrz-models.zip -d ~/audacity-openvino/openvino-models
```
4. Excluir o repositório clonado após a extração:
```bash
rm -rf whisper.cpp-openvino-models
```
## Separação de Música
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```
2. Clonar o repositório do Hugging Face:
```bash
git clone https://huggingface.co/Intel/demucs-openvino
```
3. Copiar os arquivos IR do Demucs OpenVINO:
```bash
cp demucs-openvino/htdemucs_v4.bin ~/audacity-openvino/openvino-models
cp demucs-openvino/htdemucs_v4.xml ~/audacity-openvino/openvino-models
```
4. Excluir o repositório clonado após a extração:
```bash
rm -rf demucs-openvino
```

## Supressão de Ruído
1. Vamos para o diretorio `openvino-models`:
```bash
cd ~/audacity-openvino/openvino-models
```
2. Clonar o repositório DeepFilterNet no Hugging Face:
```bash
git clone https://huggingface.co/Intel/deepfilternet-openvino
```
3. Extrair os modelos DeepFilterNet2:
```bash
unzip deepfilternet-openvino/deepfilternet2.zip -d ~/audacity-openvino/openvino-models
```
4. Extrair os modelos DeepFilterNet3:
```bash
unzip deepfilternet-openvino/deepfilternet3.zip -d ~/audacity-openvino/openvino-models
```
5. Retornamos ao diretorio `openvino-models`:
```bash
cd ~/audacity-openvino/openvino-models
```
6. Baixar os arquivos IR do modelo `noise-suppression-denseunet-ll-0001`:
```bash
wget -vc https://storage.openvinotoolkit.org/repositories/open_model_zoo/2023.0/models_bin/1/noise-suppression-denseunet-ll-0001/FP16/noise-suppression-denseunet-ll-0001.xml
```
```bash
wget -vc https://storage.openvinotoolkit.org/repositories/open_model_zoo/2023.0/models_bin/1/noise-suppression-denseunet-ll-0001/FP16/noise-suppression-denseunet-ll-0001.bin
```

Passo 16: Copiando superestrutura
---
1. Vamos retornar novamente para o diretorio `openvino-models`:
```bash
cd ~/audacity-openvino/openvino-models
```
2. Vamos olhar a arvore de diretórios:
```bash
tree -d
```
3. Após realizar todos os passos, a estrutura funcional do diretório `openvino-models` ficará assim:
```
.
├── deepfilternet2
├── deepfilternet3
├── deepfilternet-openvino
└── musicgen
    ├── mono
    └── stereo

7 directories

```
4. Para termos uma ideia melhor da disposição geral dos arquivos/pastas e seus tamanhos, executemos:
```bash
tree -h
```
E então teremos uma ideia geral mais abrangente:
```
[ 830]  .
├── [ 112]  deepfilternet2
│   ├── [3.2M]  df_dec.bin
│   ├── [112K]  df_dec.xml
│   ├── [2.5M]  enc.bin
│   ├── [175K]  enc.xml
│   ├── [3.2M]  erb_dec.bin
│   └── [181K]  erb_dec.xml
├── [ 112]  deepfilternet3
│   ├── [3.2M]  df_dec.bin
│   ├── [123K]  df_dec.xml
│   ├── [1.8M]  enc.bin
│   ├── [186K]  enc.xml
│   ├── [3.1M]  erb_dec.bin
│   └── [185K]  erb_dec.xml
├── [ 126]  deepfilternet-openvino
│   ├── [8.2M]  deepfilternet2.zip
│   ├── [7.6M]  deepfilternet3.zip
│   └── [2.1K]  README.md
├── [141M]  ggml-base.bin
├── [ 39M]  ggml-base-encoder-openvino.bin
├── [281K]  ggml-base-encoder-openvino.xml
├── [465M]  ggml-small.bin
├── [168M]  ggml-small-encoder-openvino.bin
├── [804K]  ggml-small-encoder-openvino.xml
├── [465M]  ggml-small.en-tdrz.bin
├── [168M]  ggml-small.en-tdrz-encoder-openvino.bin
├── [512K]  ggml-small.en-tdrz-encoder-openvino.xml
├── [ 96M]  htdemucs_v4.bin
├── [1.8M]  htdemucs_v4.xml
├── [ 610]  musicgen
│   ├── [1.9M]  attention_mask_from_prepare_4d_causal_10s.raw
│   ├── [492K]  attention_mask_from_prepare_4d_causal_5s.raw
│   ├── [258K]  encodec_20s.xml
│   ├── [258K]  encodec_5s.xml
│   ├── [ 56M]  encodec_combined_weights.bin
│   ├── [441K]  encodec_encoder_10s.xml
│   ├── [441K]  encodec_encoder_5s.xml
│   ├── [ 56M]  encodec_encoder_combined_weights.bin
│   ├── [ 978]  mono
│   │   ├── [ 16M]  embed_tokens.bin
│   │   ├── [ 14K]  embed_tokens.xml
│   │   ├── [3.0M]  enc_to_dec_proj.bin
│   │   ├── [2.7K]  enc_to_dec_proj.xml
│   │   ├── [ 96M]  initial_cross_attn_kv_producer.bin
│   │   ├── [173K]  initial_cross_attn_kv_producer.xml
│   │   ├── [ 16M]  lm_heads.bin
│   │   ├── [ 11K]  lm_heads.xml
│   │   ├── [672M]  musicgen_decoder_combined_weights.bin
│   │   ├── [337M]  musicgen_decoder_combined_weights_int8.bin
│   │   ├── [2.5M]  musicgen_decoder_static0_10s.xml
│   │   ├── [2.5M]  musicgen_decoder_static0_5s.xml
│   │   ├── [3.0M]  musicgen_decoder_static_batch1_int8.xml
│   │   ├── [2.5M]  musicgen_decoder_static_batch1.xml
│   │   ├── [3.0M]  musicgen_decoder_static_int8.xml
│   │   ├── [2.5M]  musicgen_decoder_static.xml
│   │   └── [8.0M]  sinusoidal_positional_embedding_weights_2048_1024.raw
│   ├── [775K]  musicgen-small-tokenizer.bin
│   ├── [5.7K]  musicgen-small-tokenizer.xml
│   ├── [ 978]  stereo
│   │   ├── [ 32M]  embed_tokens.bin
│   │   ├── [ 28K]  embed_tokens.xml
│   │   ├── [3.0M]  enc_to_dec_proj.bin
│   │   ├── [2.7K]  enc_to_dec_proj.xml
│   │   ├── [192M]  initial_cross_attn_kv_producer.bin
│   │   ├── [145K]  initial_cross_attn_kv_producer.xml
│   │   ├── [ 32M]  lm_heads.bin
│   │   ├── [ 21K]  lm_heads.xml
│   │   ├── [672M]  musicgen_decoder_combined_weights.bin
│   │   ├── [337M]  musicgen_decoder_combined_weights_int8.bin
│   │   ├── [2.5M]  musicgen_decoder_static0_10s.xml
│   │   ├── [2.5M]  musicgen_decoder_static0_5s.xml
│   │   ├── [3.0M]  musicgen_decoder_static_batch1_int8.xml
│   │   ├── [2.5M]  musicgen_decoder_static_batch1.xml
│   │   ├── [3.0M]  musicgen_decoder_static_int8.xml
│   │   ├── [2.5M]  musicgen_decoder_static.xml
│   │   └── [8.0M]  sinusoidal_positional_embedding_weights_2048_1024.raw
│   ├── [209M]  t5.bin
│   └── [550K]  t5.xml
├── [8.2M]  noise-suppression-denseunet-ll-0001.bin
└── [674K]  noise-suppression-denseunet-ll-0001.xml

7 directories, 74 files
```
Se a estrutura acima é exatamente o que você tem, então está tudo preparado.  

5. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```

6. Todos estes arquivos em `openvino-models` terão de ser transferidos para `/usr/local/lib`, então execute:
```bash
sudo cp -R openvino-models /usr/local/lib/
```

Colocando atalho no menu do sistema
---
Conforme mencionado antes, o Audacity só funcionará de acordo com o OpenVINO se o executar a partir do terminal, assim:
```bash
$HOME/audacity-openvino/audacity-build/Release/bin/audacity
```
E não rodará diretamente do seu ambiente gráfico, mas consegui um 'workaround' que resolverá essa situação, se você quiser prosseguir então use ou instale o programa chamado 'menulibre', ele permite acrescentar ou editar o menu do seu ambiente gráfico, ele usa os padrões XDG então não importa muito qual ambiente grafico esteje usando, irá funcionar.

Vamos isntalar o menulibre, caso ele ainda não exista em seu sistema:
```bash
sudo apt install -y menulibre
```
Depois carregue o menulibre, e procure pelo 'Audacity', caso não haja nenhuma entrada para Audacity, então crie-o. 
Geralmente eu acho tal atalho na seção 'Multimedia', se encontrar, troque: 
```
env GDK_BACKEND=x11 UBUNTU_MENUPROXY=0 audacity %F
```
por  
```
$HOME/audacity-openvino/audacity-build/Release/bin/audacity
```
Os gestores de menu de ambiente gráfico não gostam de variaveis como **$HOME**, então sempre que possivel troque-a por **/home/fulano**, onde *fulano* é o seu *login*.  
Também é imprescindivel marcar a opção **Executar pelo terminal** senão, não funcionará. 


Conclusões finais
--
Seguindo esses passos, o plugin Intel OpenVINO estará instalado e funcionando no Audacity 3.7 no Ubuntu 24.04.
Eu fiz o teste e garanto o resultado até data em que este artigo foi concluído.
Gostaria de informar também que este artigo foi baseado num outro HowTo:
[Audacity OpenVINO module build for Linux (Ubuntu 22.04)](https://github.com/intel/openvino-plugins-ai-audacity/blob/main/doc/build_doc/linux/README.md)
