# Automacao
Projeto de Comunicação Master-Slave entre PLCs S7-1500 TIA Portal SIEMENS


# Projeto de Comunicação Master-Slave entre PLCs S71500 TIA Portal SIEMENS

Este repositório contém o projeto completo de comunicação Master-Slave entre PLCs S71500 usando o TIA Portal SIEMENS.

## Descrição do Projeto

O projeto implementa uma comunicação cíclica entre um PLC mestre e um ou mais PLCs escravos. Os PLCs utilizam os blocos de função `TSEND_C` e `TRCV_C` para enviar e receber dados. O projeto inclui a configuração de arrays para instâncias de FBs `TSEND_C` e `TRCV_C`, e a implementação de lógica para monitorar e confirmar a recepção de bits.

## Configuração

### IPs e Portas
- **PLC Mestre**
  - IP: `192.168.1.10`
  - Porta `send`: `4000`
  - Porta `receive`: `4001`
  - ID `send`: `100`
  - ID `receive`: `101`
- **PLC Escravo**
  - IP: `192.168.1.5`
  - Porta `send`: `4001`
  - Porta `receive`: `4000`
  - ID `send`: `101`
  - ID `receive`: `100`

### Estrutura do Projeto

#### UDTs (User Defined Types)
- **UDT_DataReceive**
  - Campos: `bit`, `int`, `real`, `word`, `dword`, `dint` (todos arrays de 100 elementos)

- **UDT_DataTransmit**
  - Campos: `bit`, `int`, `real`, `word`, `dword`, `dint` (todos arrays de 100 elementos)

#### Blocos de Função
- **FB_MasterComm**
  - Configurado para iterar sobre os arrays de dados e enviar/receber bits de vida (`LifeBit`) entre o mestre e os escravos.

- **FB_SlaveComm**
  - Configurado para receber dados do mestre e enviar de volta a confirmação de recepção.

### Código do PLC Mestre

```plaintext
FOR #i := 0 TO 4 DO
    // Configura TRCV_C para recepção de dados
    #PLC_RCV[#i](EN_R := #RCV[#i].enable,
                 CONT := #RCV[#i].contRcv,
                 LEN := 0,
                 ADHOC := TRUE,
                 DONE => #RCV[#i].doneRcv,
                 BUSY => #RCV[#i].busyRcv,
                 ERROR => #RCV[#i].errorRcv,
                 STATUS => #RCV[#i].statusRcv,
                 RCVD_LEN => #RCV[#i].rcvdLen,
                 CONNECT := #RCV[#i].connectRcv,
                 DATA := #DATA_RCV[#i],
                 ADDR := NULL,
                 COM_RST := "AlwaysFALSE");
    
    // Lógica para definir sendReq baseado no Clock_2Hz
    IF #Clock_2Hz AND #SEND[#i].contSend AND NOT #SEND[#i].sendReq THEN
        #SEND[#i].sendReq := TRUE; // Define sendReq como TRUE para iniciar o envio de dados
    END_IF;
    
    // Configura TSEND_C para envio de dados
    #PLC_SEND[#i](REQ := #SEND[#i].sendReq,
                  CONT := #SEND[#i].contSend,
                  LEN := 0,
                  DONE => #SEND[#i].doneSend,
                  BUSY => #SEND[#i].busySend,
                  ERROR => #SEND[#i].errorSend,
                  STATUS => #SEND[#i].statusSend,
                  CONNECT := #SEND[#i].connectSend,
                  DATA := #DATA_SEND[#i],
                  ADDR := NULL,
                  COM_RST := "AlwaysFALSE");
    
    // Lógica para redefinir SEND.sendReq
    IF NOT #SEND[#i].busySend THEN
        #SEND[#i].sendReq := FALSE; // Redefine sendReq como FALSE se o envio não estiver em andamento
    END_IF;
    
    // Inicializa as variáveis no primeiro ciclo de execução
    IF "FirstScan" THEN
        #RCV[#i].enable := TRUE;
        #SEND[#i].life_bit := FALSE;
        #RCV[#i].life_bit := FALSE;
        #RCV[#i].Falha_Comunicacao := FALSE;
        #SEND[#i].Falha_Comunicacao := FALSE;
        #RCV[#i].CommunicationActive := FALSE;
        #SEND[#i].CommunicationActive := FALSE;
        "FirstScan" := FALSE;
    END_IF;
    
    // Temporizador TON para gerar o bit de envio
    #RCV[#i].life_time.TON(IN := #RCV[#i].enable,
                           PT := T#1S,
                           Q => #RCV[#i].life_bit);
    
    // Quando o bit de envio está ativo, enviar o bit [0]
    IF #RCV[#i].life_bit THEN
        #DATA_SEND[#i].bit[0] := TRUE;
    END_IF;
    
    // Temporizador TOF para monitorar a confirmação do bit enviado
    #RCV[#i].life_time.TOF(IN := NOT #DATA_RCV[#i].bit[0],
                           PT := T#10S,
                           Q => #RCV[#i].life_bit);
    
    // Verifica a transição do bit de recepção de 0 para 1
    IF #DATA_RCV[#i].bit[0] AND (NOT #Last_Data_RCV_Bit[#i]) THEN
        // Para o temporizador TON
        #RCV[#i].life_time.IN := FALSE;
        // Reseta o temporizador TON
        RESET_TIMER(#RCV[#i].life_time);
        
        #DATA_SEND[#i].bit[0] := FALSE; // Reseta o bit de envio
    END_IF;
    
    // Atualiza o último estado do bit de recepção [0]
    #Last_Data_RCV_Bit[#i] := #DATA_RCV[#i].bit[0];
    
    // Indica falha de comunicação se o temporizador TOF expirar ou se houver erro
    #RCV[#i].Falha_Comunicacao := NOT #RCV[#i].life_bit OR #RCV[#i].errorRcv OR #SEND[#i].errorSend;
    #SEND[#i].Falha_Comunicacao := NOT #RCV[#i].life_bit OR #RCV[#i].errorRcv OR #SEND[#i].errorSend;
    
    // Define CommunicationActive como TRUE se não houver falhas
    IF NOT #RCV[#i].Falha_Comunicacao AND NOT #SEND[#i].Falha_Comunicacao THEN
        #RCV[#i].CommunicationActive := TRUE;
        #SEND[#i].CommunicationActive := TRUE;
    ELSE
        #RCV[#i].CommunicationActive := FALSE;
        #SEND[#i].CommunicationActive := FALSE;
    END_IF;
    
    // Temporizador para monitorar se o bit de envio permanece em 1 por mais de 10 segundos
    #SEND[#i].send_timeout.TON(IN := #DATA_SEND[#i].bit[0],
                               PT := T#10S,
                               Q => #RCV[#i].Falha_Comunicacao);
    
    IF #SEND[#i].send_timeout.Q THEN
        #RCV[#i].Falha_Comunicacao := TRUE; // Gera falha de comunicação
        #SEND[#i].Falha_Comunicacao := TRUE; // Gera falha de comunicação
    END_IF;
    
    // Lógica adicional para tratar falhas de comunicação
    IF #RCV[#i].Falha_Comunicacao OR #SEND[#i].Falha_Comunicacao THEN
        // Reinicializa variáveis e tenta restabelecer a comunicação
        #RCV[#i].enable := FALSE;
        #SEND[#i].life_bit := FALSE;
        #RCV[#i].life_bit := FALSE;
        #RCV[#i].CommunicationActive := FALSE;
        #SEND[#i].CommunicationActive := FALSE;
        // Após um breve período, habilitar novamente
        #RCV[#i].enable := TRUE;
    END_IF;
END_FOR;





Código do PLC Escravo




// Configura TRCV_C para recepção de dados
"TRCV_C_DB"(EN_R := #RCV.enable,
            CONT := #RCV.contRcv,
            LEN := 0,
            ADHOC := "AlwaysTRUE",
            DONE => #RCV.doneRcv,
            BUSY => #RCV.busyRcv,
            ERROR => #RCV.errorRcv,
            STATUS => #RCV.statusRcv,
            RCVD_LEN := #RCV.rcvdLen,
            CONNECT := #RCV.connectRcv,
            DATA := #DATA_RCV,
            ADDR := NULL,
            COM_RST := "AlwaysFALSE");

// Lógica para definir sendReq baseado no Clock_2Hz
IF #Clock_2Hz AND #SEND.contSend AND NOT #SEND.sendReq THEN
    #SEND.sendReq := TRUE; // Define sendReq como TRUE para iniciar o envio de dados
END_IF;

// Configura TSEND_C para envio de dados
"TSEND_C_DB"(REQ := #SEND.sendReq,
             CONT := #SEND.contSend,
             LEN := 0,
             DONE := #SEND.doneSend,
             BUSY := #SEND.busySend,
             ERROR := #SEND.errorSend,
             STATUS := #SEND.statusSend,
             CONNECT := #SEND.connectSend,
             DATA := #DATA_SEND,
             ADDR := NULL,
             COM_RST := "AlwaysFALSE");

// Lógica para redefinir SEND.sendReq
IF NOT #SEND.busySend THEN
    #SEND.sendReq := FALSE; // Redefine sendReq como FALSE se o envio não estiver em andamento
END_IF;

// Verifica se o bit foi recebido
IF #DATA_RCV.bit[0] THEN
    // Confirma a recepção enviando de volta o bit
    #DATA_SEND.bit[0] := TRUE;
ELSE
    // Reseta o bit de envio
    #DATA_SEND.bit[0] := FALSE;
END_IF;






Claro, aqui está o conteúdo completo para o arquivo README.md que você pode criar e adicionar ao seu repositório no GitHub.

Passo a Passo:
Crie um novo arquivo chamado README.md no seu repositório.

Cole o seguinte conteúdo no arquivo README.md:

markdown
Copiar código
# Projeto de Comunicação Master-Slave entre PLCs S71500 TIA Portal SIEMENS

Este repositório contém o projeto completo de comunicação Master-Slave entre PLCs S71500 usando o TIA Portal SIEMENS.

## Descrição do Projeto

O projeto implementa uma comunicação cíclica entre um PLC mestre e um ou mais PLCs escravos. Os PLCs utilizam os blocos de função `TSEND_C` e `TRCV_C` para enviar e receber dados. O projeto inclui a configuração de arrays para instâncias de FBs `TSEND_C` e `TRCV_C`, e a implementação de lógica para monitorar e confirmar a recepção de bits.

## Configuração

### IPs e Portas
- **PLC Mestre**
  - IP: `192.168.1.10`
  - Porta `send`: `4000`
  - Porta `receive`: `4001`
  - ID `send`: `100`
  - ID `receive`: `101`
- **PLC Escravo**
  - IP: `192.168.1.5`
  - Porta `send`: `4001`
  - Porta `receive`: `4000`
  - ID `send`: `101`
  - ID `receive`: `100`

### Estrutura do Projeto

#### UDTs (User Defined Types)
- **UDT_DataReceive**
  - Campos: `bit`, `int`, `real`, `word`, `dword`, `dint` (todos arrays de 100 elementos)

- **UDT_DataTransmit**
  - Campos: `bit`, `int`, `real`, `word`, `dword`, `dint` (todos arrays de 100 elementos)

#### Blocos de Função
- **FB_MasterComm**
  - Configurado para iterar sobre os arrays de dados e enviar/receber bits de vida (`LifeBit`) entre o mestre e os escravos.

- **FB_SlaveComm**
  - Configurado para receber dados do mestre e enviar de volta a confirmação de recepção.

### Código do PLC Mestre

```plaintext
FOR #i := 0 TO 4 DO
    // Configura TRCV_C para recepção de dados
    #PLC_RCV[#i](EN_R := #RCV[#i].enable,
                 CONT := #RCV[#i].contRcv,
                 LEN := 0,
                 ADHOC := TRUE,
                 DONE => #RCV[#i].doneRcv,
                 BUSY => #RCV[#i].busyRcv,
                 ERROR => #RCV[#i].errorRcv,
                 STATUS => #RCV[#i].statusRcv,
                 RCVD_LEN => #RCV[#i].rcvdLen,
                 CONNECT := #RCV[#i].connectRcv,
                 DATA := #DATA_RCV[#i],
                 ADDR := NULL,
                 COM_RST := "AlwaysFALSE");
    
    // Lógica para definir sendReq baseado no Clock_2Hz
    IF #Clock_2Hz AND #SEND[#i].contSend AND NOT #SEND[#i].sendReq THEN
        #SEND[#i].sendReq := TRUE; // Define sendReq como TRUE para iniciar o envio de dados
    END_IF;
    
    // Configura TSEND_C para envio de dados
    #PLC_SEND[#i](REQ := #SEND[#i].sendReq,
                  CONT := #SEND[#i].contSend,
                  LEN := 0,
                  DONE => #SEND[#i].doneSend,
                  BUSY => #SEND[#i].busySend,
                  ERROR => #SEND[#i].errorSend,
                  STATUS => #SEND[#i].statusSend,
                  CONNECT := #SEND[#i].connectSend,
                  DATA := #DATA_SEND[#i],
                  ADDR := NULL,
                  COM_RST := "AlwaysFALSE");
    
    // Lógica para redefinir SEND.sendReq
    IF NOT #SEND[#i].busySend THEN
        #SEND[#i].sendReq := FALSE; // Redefine sendReq como FALSE se o envio não estiver em andamento
    END_IF;
    
    // Inicializa as variáveis no primeiro ciclo de execução
    IF "FirstScan" THEN
        #RCV[#i].enable := TRUE;
        #SEND[#i].life_bit := FALSE;
        #RCV[#i].life_bit := FALSE;
        #RCV[#i].Falha_Comunicacao := FALSE;
        #SEND[#i].Falha_Comunicacao := FALSE;
        #RCV[#i].CommunicationActive := FALSE;
        #SEND[#i].CommunicationActive := FALSE;
        "FirstScan" := FALSE;
    END_IF;
    
    // Temporizador TON para gerar o bit de envio
    #RCV[#i].life_time.TON(IN := #RCV[#i].enable,
                           PT := T#1S,
                           Q => #RCV[#i].life_bit);
    
    // Quando o bit de envio está ativo, enviar o bit [0]
    IF #RCV[#i].life_bit THEN
        #DATA_SEND[#i].bit[0] := TRUE;
    END_IF;
    
    // Temporizador TOF para monitorar a confirmação do bit enviado
    #RCV[#i].life_time.TOF(IN := NOT #DATA_RCV[#i].bit[0],
                           PT := T#10S,
                           Q => #RCV[#i].life_bit);
    
    // Verifica a transição do bit de recepção de 0 para 1
    IF #DATA_RCV[#i].bit[0] AND (NOT #Last_Data_RCV_Bit[#i]) THEN
        // Para o temporizador TON
        #RCV[#i].life_time.IN := FALSE;
        // Reseta o temporizador TON
        RESET_TIMER(#RCV[#i].life_time);
        
        #DATA_SEND[#i].bit[0] := FALSE; // Reseta o bit de envio
    END_IF;
    
    // Atualiza o último estado do bit de recepção [0]
    #Last_Data_RCV_Bit[#i] := #DATA_RCV[#i].bit[0];
    
    // Indica falha de comunicação se o temporizador TOF expirar ou se houver erro
    #RCV[#i].Falha_Comunicacao := NOT #RCV[#i].life_bit OR #RCV[#i].errorRcv OR #SEND[#i].errorSend;
    #SEND[#i].Falha_Comunicacao := NOT #RCV[#i].life_bit OR #RCV[#i].errorRcv OR #SEND[#i].errorSend;
    
    // Define CommunicationActive como TRUE se não houver falhas
    IF NOT #RCV[#i].Falha_Comunicacao AND NOT #SEND[#i].Falha_Comunicacao THEN
        #RCV[#i].CommunicationActive := TRUE;
        #SEND[#i].CommunicationActive := TRUE;
    ELSE
        #RCV[#i].CommunicationActive := FALSE;
        #SEND[#i].CommunicationActive := FALSE;
    END_IF;
    
    // Temporizador para monitorar se o bit de envio permanece em 1 por mais de 10 segundos
    #SEND[#i].send_timeout.TON(IN := #DATA_SEND[#i].bit[0],
                               PT := T#10S,
                               Q => #RCV[#i].Falha_Comunicacao);
    
    IF #SEND[#i].send_timeout.Q THEN
        #RCV[#i].Falha_Comunicacao := TRUE; // Gera falha de comunicação
        #SEND[#i].Falha_Comunicacao := TRUE; // Gera falha de comunicação
    END_IF;
    
    // Lógica adicional para tratar falhas de comunicação
    IF #RCV[#i].Falha_Comunicacao OR #SEND[#i].Falha_Comunicacao THEN
        // Reinicializa variáveis e tenta restabelecer a comunicação
        #RCV[#i].enable := FALSE;
        #SEND[#i].life_bit := FALSE;
        #RCV[#i].life_bit := FALSE;
        #RCV[#i].CommunicationActive := FALSE;
        #SEND[#i].CommunicationActive := FALSE;
        // Após um breve período, habilitar novamente
        #RCV[#i].enable := TRUE;
    END_IF;
END_FOR;
Código do PLC Escravo
plaintext
Copiar código
// Configura TRCV_C para recepção de dados
"TRCV_C_DB"(EN_R := #RCV.enable,
            CONT := #RCV.contRcv,
            LEN := 0,
            ADHOC := "AlwaysTRUE",
            DONE => #RCV.doneRcv,
            BUSY => #RCV.busyRcv,
            ERROR => #RCV.errorRcv,
            STATUS => #RCV.statusRcv,
            RCVD_LEN := #RCV.rcvdLen,
            CONNECT := #RCV.connectRcv,
            DATA := #DATA_RCV,
            ADDR := NULL,
            COM_RST := "AlwaysFALSE");

// Lógica para definir sendReq baseado no Clock_2Hz
IF #Clock_2Hz AND #SEND.contSend AND NOT #SEND.sendReq THEN
    #SEND.sendReq := TRUE; // Define sendReq como TRUE para iniciar o envio de dados
END_IF;

// Configura TSEND_C para envio de dados
"TSEND_C_DB"(REQ := #SEND.sendReq,
             CONT := #SEND.contSend,
             LEN := 0,
             DONE := #SEND.doneSend,
             BUSY := #SEND.busySend,
             ERROR := #SEND.errorSend,
             STATUS := #SEND.statusSend,
             CONNECT := #SEND.connectSend,
             DATA := #DATA_SEND,
             ADDR := NULL,
             COM_RST := "AlwaysFALSE");

// Lógica para redefinir SEND.sendReq
IF NOT #SEND.busySend THEN
    #SEND.sendReq := FALSE; // Redefine sendReq como FALSE se o envio não estiver em andamento
END_IF;

// Verifica se o bit foi recebido
IF #DATA_RCV.bit[0] THEN
    // Confirma a recepção enviando de volta o bit
    #DATA_SEND.bit[0] := TRUE;
ELSE
    // Reseta o bit de envio
    #DATA_SEND.bit[0] := FALSE;
END_IF;
Como Utilizar
Configure os IPs e Portas conforme a tabela acima.
Carregue os blocos de função FB_MasterComm e FB_SlaveComm nos PLCs mestre e escravo respectivamente.
Certifique-se de que as UDTs UDT_DataReceive e UDT_DataTransmit estão corretamente configuradas nos PLCs.
Ajuste os IDs e as portas conforme necessário.
Inicie a comunicação cíclica e monitore os bits de vida (LifeBit) para garantir que a comunicação está ativa.







Claro, aqui está o conteúdo completo para o arquivo README.md que você pode criar e adicionar ao seu repositório no GitHub.

Passo a Passo:
Crie um novo arquivo chamado README.md no seu repositório.

Cole o seguinte conteúdo no arquivo README.md:

markdown
Copiar código
# Projeto de Comunicação Master-Slave entre PLCs S71500 TIA Portal SIEMENS

Este repositório contém o projeto completo de comunicação Master-Slave entre PLCs S71500 usando o TIA Portal SIEMENS.

## Descrição do Projeto

O projeto implementa uma comunicação cíclica entre um PLC mestre e um ou mais PLCs escravos. Os PLCs utilizam os blocos de função `TSEND_C` e `TRCV_C` para enviar e receber dados. O projeto inclui a configuração de arrays para instâncias de FBs `TSEND_C` e `TRCV_C`, e a implementação de lógica para monitorar e confirmar a recepção de bits.

## Configuração

### IPs e Portas
- **PLC Mestre**
  - IP: `192.168.1.10`
  - Porta `send`: `4000`
  - Porta `receive`: `4001`
  - ID `send`: `100`
  - ID `receive`: `101`
- **PLC Escravo**
  - IP: `192.168.1.5`
  - Porta `send`: `4001`
  - Porta `receive`: `4000`
  - ID `send`: `101`
  - ID `receive`: `100`

### Estrutura do Projeto

#### UDTs (User Defined Types)
- **UDT_DataReceive**
  - Campos: `bit`, `int`, `real`, `word`, `dword`, `dint` (todos arrays de 100 elementos)

- **UDT_DataTransmit**
  - Campos: `bit`, `int`, `real`, `word`, `dword`, `dint` (todos arrays de 100 elementos)

#### Blocos de Função
- **FB_MasterComm**
  - Configurado para iterar sobre os arrays de dados e enviar/receber bits de vida (`LifeBit`) entre o mestre e os escravos.

- **FB_SlaveComm**
  - Configurado para receber dados do mestre e enviar de volta a confirmação de recepção.

### Código do PLC Mestre

```plaintext
FOR #i := 0 TO 4 DO
    // Configura TRCV_C para recepção de dados
    #PLC_RCV[#i](EN_R := #RCV[#i].enable,
                 CONT := #RCV[#i].contRcv,
                 LEN := 0,
                 ADHOC := TRUE,
                 DONE => #RCV[#i].doneRcv,
                 BUSY => #RCV[#i].busyRcv,
                 ERROR => #RCV[#i].errorRcv,
                 STATUS => #RCV[#i].statusRcv,
                 RCVD_LEN => #RCV[#i].rcvdLen,
                 CONNECT := #RCV[#i].connectRcv,
                 DATA := #DATA_RCV[#i],
                 ADDR := NULL,
                 COM_RST := "AlwaysFALSE");
    
    // Lógica para definir sendReq baseado no Clock_2Hz
    IF #Clock_2Hz AND #SEND[#i].contSend AND NOT #SEND[#i].sendReq THEN
        #SEND[#i].sendReq := TRUE; // Define sendReq como TRUE para iniciar o envio de dados
    END_IF;
    
    // Configura TSEND_C para envio de dados
    #PLC_SEND[#i](REQ := #SEND[#i].sendReq,
                  CONT := #SEND[#i].contSend,
                  LEN := 0,
                  DONE => #SEND[#i].doneSend,
                  BUSY => #SEND[#i].busySend,
                  ERROR => #SEND[#i].errorSend,
                  STATUS => #SEND[#i].statusSend,
                  CONNECT := #SEND[#i].connectSend,
                  DATA := #DATA_SEND[#i],
                  ADDR := NULL,
                  COM_RST := "AlwaysFALSE");
    
    // Lógica para redefinir SEND.sendReq
    IF NOT #SEND[#i].busySend THEN
        #SEND[#i].sendReq := FALSE; // Redefine sendReq como FALSE se o envio não estiver em andamento
    END_IF;
    
    // Inicializa as variáveis no primeiro ciclo de execução
    IF "FirstScan" THEN
        #RCV[#i].enable := TRUE;
        #SEND[#i].life_bit := FALSE;
        #RCV[#i].life_bit := FALSE;
        #RCV[#i].Falha_Comunicacao := FALSE;
        #SEND[#i].Falha_Comunicacao := FALSE;
        #RCV[#i].CommunicationActive := FALSE;
        #SEND[#i].CommunicationActive := FALSE;
        "FirstScan" := FALSE;
    END_IF;
    
    // Temporizador TON para gerar o bit de envio
    #RCV[#i].life_time.TON(IN := #RCV[#i].enable,
                           PT := T#1S,
                           Q => #RCV[#i].life_bit);
    
    // Quando o bit de envio está ativo, enviar o bit [0]
    IF #RCV[#i].life_bit THEN
        #DATA_SEND[#i].bit[0] := TRUE;
    END_IF;
    
    // Temporizador TOF para monitorar a confirmação do bit enviado
    #RCV[#i].life_time.TOF(IN := NOT #DATA_RCV[#i].bit[0],
                           PT := T#10S,
                           Q => #RCV[#i].life_bit);
    
    // Verifica a transição do bit de recepção de 0 para 1
    IF #DATA_RCV[#i].bit[0] AND (NOT #Last_Data_RCV_Bit[#i]) THEN
        // Para o temporizador TON
        #RCV[#i].life_time.IN := FALSE;
        // Reseta o temporizador TON
        RESET_TIMER(#RCV[#i].life_time);
        
        #DATA_SEND[#i].bit[0] := FALSE; // Reseta o bit de envio
    END_IF;
    
    // Atualiza o último estado do bit de recepção [0]
    #Last_Data_RCV_Bit[#i] := #DATA_RCV[#i].bit[0];
    
    // Indica falha de comunicação se o temporizador TOF expirar ou se houver erro
    #RCV[#i].Falha_Comunicacao := NOT #RCV[#i].life_bit OR #RCV[#i].errorRcv OR #SEND[#i].errorSend;
    #SEND[#i].Falha_Comunicacao := NOT #RCV[#i].life_bit OR #RCV[#i].errorRcv OR #SEND[#i].errorSend;
    
    // Define CommunicationActive como TRUE se não houver falhas
    IF NOT #RCV[#i].Falha_Comunicacao AND NOT #SEND[#i].Falha_Comunicacao THEN
        #RCV[#i].CommunicationActive := TRUE;
        #SEND[#i].CommunicationActive := TRUE;
    ELSE
        #RCV[#i].CommunicationActive := FALSE;
        #SEND[#i].CommunicationActive := FALSE;
    END_IF;
    
    // Temporizador para monitorar se o bit de envio permanece em 1 por mais de 10 segundos
    #SEND[#i].send_timeout.TON(IN := #DATA_SEND[#i].bit[0],
                               PT := T#10S,
                               Q => #RCV[#i].Falha_Comunicacao);
    
    IF #SEND[#i].send_timeout.Q THEN
        #RCV[#i].Falha_Comunicacao := TRUE; // Gera falha de comunicação
        #SEND[#i].Falha_Comunicacao := TRUE; // Gera falha de comunicação
    END_IF;
    
    // Lógica adicional para tratar falhas de comunicação
    IF #RCV[#i].Falha_Comunicacao OR #SEND[#i].Falha_Comunicacao THEN
        // Reinicializa variáveis e tenta restabelecer a comunicação
        #RCV[#i].enable := FALSE;
        #SEND[#i].life_bit := FALSE;
        #RCV[#i].life_bit := FALSE;
        #RCV[#i].CommunicationActive := FALSE;
        #SEND[#i].CommunicationActive := FALSE;
        // Após um breve período, habilitar novamente
        #RCV[#i].enable := TRUE;
    END_IF;
END_FOR;
Código do PLC Escravo
plaintext
Copiar código
// Configura TRCV_C para recepção de dados
"TRCV_C_DB"(EN_R := #RCV.enable,
            CONT := #RCV.contRcv,
            LEN := 0,
            ADHOC := "AlwaysTRUE",
            DONE => #RCV.doneRcv,
            BUSY => #RCV.busyRcv,
            ERROR => #RCV.errorRcv,
            STATUS => #RCV.statusRcv,
            RCVD_LEN := #RCV.rcvdLen,
            CONNECT := #RCV.connectRcv,
            DATA := #DATA_RCV,
            ADDR := NULL,
            COM_RST := "AlwaysFALSE");

// Lógica para definir sendReq baseado no Clock_2Hz
IF #Clock_2Hz AND #SEND.contSend AND NOT #SEND.sendReq THEN
    #SEND.sendReq := TRUE; // Define sendReq como TRUE para iniciar o envio de dados
END_IF;

// Configura TSEND_C para envio de dados
"TSEND_C_DB"(REQ := #SEND.sendReq,
             CONT := #SEND.contSend,
             LEN := 0,
             DONE := #SEND.doneSend,
             BUSY := #SEND.busySend,
             ERROR := #SEND.errorSend,
             STATUS := #SEND.statusSend,
             CONNECT := #SEND.connectSend,
             DATA := #DATA_SEND,
             ADDR := NULL,
             COM_RST := "AlwaysFALSE");

// Lógica para redefinir SEND.sendReq
IF NOT #SEND.busySend THEN
    #SEND.sendReq := FALSE; // Redefine sendReq como FALSE se o envio não estiver em andamento
END_IF;

// Verifica se o bit foi recebido
IF #DATA_RCV.bit[0] THEN
    // Confirma a recepção enviando de volta o bit
    #DATA_SEND.bit[0] := TRUE;
ELSE
    // Reseta o bit de envio
    #DATA_SEND.bit[0] := FALSE;
END_IF;
Como Utilizar
Configure os IPs e Portas conforme a tabela acima.
Carregue os blocos de função FB_MasterComm e FB_SlaveComm nos PLCs mestre e escravo respectivamente.
Certifique-se de que as UDTs UDT_DataReceive e UDT_DataTransmit estão corretamente configuradas nos PLCs.
Ajuste os IDs e as portas conforme necessário.
Inicie a comunicação cíclica e monitore os bits de vida (LifeBit) para garantir que a comunicação está ativa.


Contato
Danilo Lira
Localização: Lisboa, Portugal
Telefone: +351 935 479 757
LinkedIn: Danilo Lira
Empresa: RLS Automação Industrial, Sintra

Este projeto foi desenvolvido para facilitar a comunicação entre PLCs Siemens utilizando o TIA Portal, proporcionando uma base sólida para projetos de automação industrial.

