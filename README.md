
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

Passo 1: Tenho uma GPU?
---
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

Passo 2: Tenho OpenCL instalado e funcional? 
---
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


Passo 3: Instale as dependências necessárias
---

```bash
sudo apt install -y build-essential cmake git python3-pip libgtk2.0-dev libasound2-dev libjack-jackd2-dev uuid-dev ocl-icd-opencl-dev
sudo apt install -y python3.12-venv
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
Extraia os arquivos:
```bash
tar zxvf openvino_tokenizers_ubuntu24_2024.5.0.0_x86_64.tar.gz
```
Serão extraidos os seguintes arquivos na pasta runtime/lib/intel64/ e que nos interessam:
```
(...)
runtime/lib/intel64/libcore_tokenizers.so
runtime/lib/intel64/libopenvino_tokenizers.so
```
Precisartemos copiá-los para: (cuidado, linha muito extensa)
```bash
cp -r ~/audacity-openvino/openvino_toolkit/install_dependencies/runtime/lib/intel64/* \
      ~/audacity-openvino/openvino_toolkit/runtime/lib/intel64
```
Remova o arquivo baixado:
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
sudo apt install ocl-icd-opencl-dev
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
Descompactamos o arquivo:
```bash
tar zxvf l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
```
Preparamos o ambiente:
```bash
source $HOME/audacity-openvino/l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64/setupvars.sh
```

Isso é necessário para os procedimentos a seguir, depois removemos o arquivo baixado:
```bash
rm -f l_openvino_toolkit_ubuntu24_2024.5.0.17288.7975fa5da0c_x86_64.tgz
```


Passo 11: Construção de Subcomponentes
---
1. Vamos para o diretorio 'base':
```bash
cd ~/audacity-openvino
```
Agora vamos construir o `whisper.cpp`. Primeiro vamos baixá-lo:
```bash
git clone https://github.com/ggerganov/whisper.cpp
```
Depois estabelecer a versão que desejamos do whisper.cpp:
```bash
cd whisper.cpp
git checkout v1.5.4
```
E agora, vamos criar pasta de construção e compilá-lo:
```bash
cd ..
mkdir whisper-build
cd whisper-build
```
Vamos agora rodar o cmake, especificadno que desejamos habiitar o suporte ao OpenVINO:
```bash
cmake ../whisper.cpp/ -DWHISPER_OPENVINO=ON
```
Agora fazemos o processo de build:
```bash
make -j$(nproc)
```
Ao invés do 'sudo make install' e fazer uma instalação global para o sistema, vamos gerar os binários em './installed':
```bash
cmake --install . --config Release --prefix ./installed
```
Notará que em './installed', os arquivos do 'whisper' que nos importam são:
```
/installed/lib/libwhisper.so
/installed/include/ggml.h
/installed/include/whisper.h
```
Se você optar por um 'sudo make install' estes arquivos estariam em:
```
/usr/local/lib/libwhisper.so
/usr/local/include/ggml.h
/usr/local/include/whisper.h
```
Com a compilação/instalação concluída, a compilação do Audacity encontrará um erro colateral compilado por meio do WHISPERCPP_ROOTDIR. Então você pode configurá-lo assim:
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
sudo apt-get install -y build-essential cmake git python3-pip
sudo apt-get install -y libgtk2.0-dev libasound2-dev libjack-jackd2-dev uuid-dev
sudo pip3 install conan
```
Vamos clonar o repositorio do Audacity:
```bash
git clone https://github.com/audacity/audacity.git
```
Vamos informar a versão desejada que iremos compilar:
```bash
cd audacity
git checkout release-3.7.0
cd ..
```
Vamos agora fazer o build fora da pasta do audacity:
```bash
cd ..
mkdir audacity-build
cd audacity-build
```
Vamos compilá-lo(vá tomar um café, porque será um pouco demorado):
```bash
cmake -G "Unix Makefiles" ../audacity -DCMAKE_BUILD_TYPE=Release
```
Depois vamos a uma construção:
```bash
make -j$(nproc)
```
Quando isso estiver feito, você pode executar o Audacity assim (do diretório audacity-build):
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
# Whisper.cpp
export LIBTORCH_ROOTDIR=$HOME/audacity-openvino/libtorch
```
6. Estamos prontos para recompilar novamente o Audacity
```bash
cd ~/audacity-openvino/audacity-build
```
7. Vamos rodar o processo de cnake novamente
```bash
cmake -G "Unix Makefiles" ../audacity -DCMAKE_BUILD_TYPE=Release
```
8. Novamente o make...
```bash
make -j$(nproc)
```
Se tudo for compilado corretamente, você verá mod-openvino.so em ./Release/lib/audacity/modules/

Executando o audacity pela primeira vez
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

Mas se quiser insistir e forçá-lo a carregar o programa de outra forma, um 'workaround' que criei poderá fazer isso funcionar, use ou instale o programa chamado 'menulibre', ele permite acrescentar ou editar o menu do ambiente gráfico, para instalar:
```bash
sudo apt install menulibre
```
Depois carregue o menulibre, e procure pelo 'Audacity', caso não haja nenhuma entrada para Audacity, então crie-o. 
Geralmente eu acho tal atalho na seção 'Multimedia', se encontrar, troque: 
```
env GDK_BACKEND=x11 UBUNTU_MENUPROXY=0 audacity %F**
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
