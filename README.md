# PROJETO DE ENCERRAMENTO DE RESIDÊNCIA
### DISPOSITIVO IoT PARA SEGURANÇA PATRIMONIAL E DETECÇÃO INTELIGENTE DE INTRUSOS

## 1. O Problema 

Sou Diretor de Segurança de uma empresa de alta tecnologia. Possuímos salas de servidores centrais, laboratórios de desenvolvimento e cofres de protótipos que são ambientes estritamente proibidos para pessoas não autorizadas. Os sistemas de segurança atuais baseados em sensores de presença comuns (PIR) falham muito, pois geram alarmes falsos com pequenas variações de luz ou animais, fazendo com que a equipe de vigilância perca a confiabilidade nos alertas. Por outro lado, manter câmeras gravando em alta resolução 100% do tempo consome muita banda de rede e armazenamento de servidores.

Precisamos de uma solução embarcada inteligente (*Edge AI*) para vigilância discreta. O dispositivo operará em modo de baixo consumo e vigilância silenciosa, utilizando um sensor de distância a laser (ToF) de alta precisão apontado para a zona de passagem. Se a distância lida diminuir abruptamente, significa que um objeto ou corpo interrompeu o feixe a laser.

A partir desse gatilho físico, o sistema deve iniciar um fluxo inteligente em duas etapas críticas para evitar loops infinitos de processamento:

* **Confirmação e Registro de Invasão:** O firmware deve ativar a câmera imediatamente e rodar um algoritmo de detecção de humanos. Caso um humano seja confirmado no ambiente, o sistema deve capturar uma sequência de 2 a 3 imagens com intervalo de 2 segundos entre elas e enviá-las imediatamente ao servidor de segurança como prova técnica da intrusão.
* **Reconhecimento Facial:** Em seguida, o sistema deve entrar em uma rotina dedicada para tentar detectar e capturar especificamente o rosto (*face*) do invasor em close-up, enviando essa imagem focalizada para o servidor.
* **Gerenciamento de Tempo (Timeout):** Para evitar que o dispositivo trave nessa rotina de busca facial infinitamente (caso o invasor fique de costas ou saia correndo), o firmware deve implementar uma lógica de tempo limite (*timeout*). Após esse tempo sem sucesso ou após concluir as capturas com sucesso, o sistema deve resetar suas variáveis e voltar ao estado de repouso, monitorando apenas o sensor laser.

## 2. Recomendações de Firmware e Conectividade

* **Lógica de Estados (FSM):** Recomenda-se fortemente projetar o firmware utilizando uma Máquina de Estados Finados. Os estados claros seriam: `AGUARDANDO_GATILHO` (lendo o sensor laser), `VERIFICANDO_HUMANO` (processando os primeiros frames), `REGISTRANDO_INTRUSO` (capturando as 3 fotos temporizadas), `BUSCANDO_ROSTO` (tentando enquadrar a face) e `TIMEOUT_RECOVERY` (tempo de resguardo antes de voltar a monitorar).
* **Temporização Não-Bloqueante:** Os intervalos de 2 segundos entre as fotos de intrusão e o timeout da busca facial não podem ser feitos com funções `delay()`. Use o gerenciamento de tempo baseado em contagem de milissegundos (`millis()` ou timers do ESP-IDF) para garantir que a câmera continue capturando frames e processando a imagem enquanto o tempo passa.
* **Configuração e Otimização da Câmera (OV5640):** A câmera OV5640 possui alta resolução, mas para os algoritmos de detecção/reconhecimento rodarem de forma eficiente no ESP32-S3, configure resoluções menores para o processamento de borda (ex: QVGA ou VGA) e utilize formatos de compressão (JPEG) para agilizar o upload das fotos capturadas para o servidor.
* **Tratamento do Sensor VL53L0X (I2C):** O sensor de distância via laser (*Time-of-Flight*) deve ser lido continuamente em modo contínuo ou temporizado. O firmware deve calcular a média das distâncias padrão da sala vazia e disparar o alarme quando houver uma queda percentual abrupta nessa distância.
* **Protocolo de Transmissão de Mídia:** Devido ao tamanho das imagens em sequência, a equipe deve avaliar a melhor estratégia de rede (ex: requisições HTTP POST multipart para envio dos arquivos JPEG diretamente para um servidor web backend, ou fragmentação em payloads binários via MQTT).

## 3. Requisitos de Modelagem 3D (Gabinete)

O design 3D deste dispositivo deve focar na discrição e no posicionamento milimétrico dos sensores:

* **Posicionamento Óptico Duplo:** O case deve possuir aberturas perfeitamente alinhadas para a lente da câmera OV5640 e também para os dois emissores/receptores ópticos do sensor laser VL53L0X. Eles não podem obstruir a visão um do outro e nem gerar reflexos internos no plástico do case.
* **Discrição e Camuflagem:** O design exterior do gabinete deve ser compacto e discreto, simulando um sensor de presença industrial ou de incêndio, permitindo que ele seja fixado no teto ou no topo do batente de uma porta sem chamar a atenção do invasor.
* **Dissipação Térmica em Processamento:** Algoritmos de visão computacional na borda demandam processamento intensivo do ESP32-S3, o que gera calor considerável. O modelo 3D deve conter ranhuras de ventilação passiva nas laterais ou na parte traseira para evitar o superaquecimento do microcontrolador durante as rotinas de análise de imagem.

## 4. Entregável Inicial: Documento de Arquitetura e Viabilidade Técnica

Antes de iniciar o desenvolvimento prático do firmware ou a modelagem 3D do gabinete, a equipe deverá entregar um relatório técnico de arquitetura contendo obrigatoriamente:

* **4.1. Mapeamento de Hardware e Componentes:** A equipe deve identificar e documentar os pinos e conexões da placa educacional:
    * **Identificação:** Modelos e especificações do microcontrolador ESP32-S3, da câmera OV5640 e do sensor laser VL53L0X.
    * **Pinout (Tabela de GPIOs):** Tabela detalhada mapeando as conexões do barramento de dados paralelo da câmera (D0-D7, VSYNC, HREF, PCLK, XCLK) e os pinos de controle (SDA/SCL para a configuração da câmera e do sensor laser).
    * **Barramentos Internos:** Detalhar o compartilhamento do barramento I2C entre os periféricos (se houver) e como os endereços de hardware do VL53L0X e do SCCB da câmera se organizam na mesma linha.

* **4.2. Diagrama de Blocos do Sistema:** Apresentar um esquema gráfico modular do hardware. No centro deve figurar o microcontrolador ESP32-S3 e, ao redor, blocos conectados por setas indicando o fluxo de sinal:
    * **Setas de entrada (In):** Sensor VL53L0X (via I2C) e Câmera OV5640 (via barramento de câmera digital) enviando informações para o ESP32-S3.
    * **Fluxo de processamento interno:** Indicação em bloco da memória PSRAM do ESP32-S3 sendo usada como buffer para as imagens da câmera.
    * **Fluxo de rede:** Seta de saída demonstrando o envio dos arquivos de imagem sequenciais e de close-up facial para o servidor central através da rede sem fio.

* **4.3. Especificação do Protocolo de Comunicação e Estratégia de Software:**
    * **Arquitetura da Máquina de Estados:** Apresentar um diagrama de fluxo ou descrição detalhada de como o firmware sairá de um estado para o outro e, principalmente, qual será o critério de *Timeout* (em segundos) adotado para desistir da busca pelo rosto do invasor.
    * **Protocolo de Aplicação de Rede:** Justificativa da escolha do protocolo para o transporte das imagens (HTTP REST vs MQTT) com base no tamanho estimado de cada arquivo gerado pela câmera.
    * **Modelagem do Payload:** Exemplo de como o servidor receberá o alerta (ex: se o envio de dados textuais com metadados do sensor irá junto com a imagem ou em pacotes separados).
