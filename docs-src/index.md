# Webserver com Servidor TFTP para automatização de deploy de aplicações na DE10-Standard

- **Alunos:** Francisco Pinheiro Janela / Wilgner  / Marco
- **Curso:** Engenharia da Computação / Mecatrônica / Mecatrônica
- **Semestre:** 6
- **Contato:** francisco.pinheiro.janela@gmail.com / placeholder@gmail.com / placeholder@gmail.com
- **Ano:** 2022.2

## Começando

Para seguir esse tutorial é necessário:

- **Hardware:** 
    - DE10-Standard
    - NUC ou máquina para server Linux
    - Roteador (*modelo usado:* tp-link TL-R470T+)
    - Switch (*modelo usado:* D-Link DGS-1100-08P)
- **Softwares:** Quartus 18.01
- **Documentos:** [DE10-Standard_User_manual.pdf](https://github.com/Insper/DE10-Standard-v.1.3.0-SystemCD/tree/master/Manual), [U-Boot Script](https://ece453.engr.wisc.edu/u-boot-script/), [U-Boot TFTP e NFS](https://www.rocketboards.org/foswiki/Documentation/BootingAlteraSoCFPGAFromNetworkUsingTFTPAndNFS)
- **Laboratórios:** `Linux embarcado`, `Configurando infra`, `Extra: configurando rede`, `Compilando o kernel` e `Buildroot`.

### Montando a Infraestrutura

Para montar a infraestrutura para este roteiro vamos precisar de um ponto de acesso à rede via cabo. Além disso, como a rede do insper é protegida, o roteador que estará constituindo nosso kit precisa estar com o MAC Adress configurado na rede interna.

Com essas condições podemos, então, ligar toda a fiação:

- `Roteador` - Porta WAN1 deve estar conectada ao ponto de acesso, a porta LAN1 ao Switch e alimentá-lo com energia

- `Switch` - Mínimo de 4 portas, deve alimentá-lo com energia

- `NUC` - Conectar o cabo LAN ao Switch, ligar com Cabo USB a NUC à DE10-Standard e alimentá-la com energia

- `DE10-Standard` - Conectar via cabo LAN ao Switch, conectar o USB da NUC, inserir o SDCard configurado com linux kernel e buildroot como filesystem (laboratórios previamente feitos) e alimentá-la com energia

Após tudo conectado, com um palito ou objeto fino, clicar nos pontos de RESET tanto do Roteador, quanto do Switch. O resultado deve ficar similar ao esquema abaixo:

![](imagem-infra.png){width=250}

### Configurando a Infraestrtura

Com a infra montada, podemos começar a configurar todos os serviços. Vamos começar pela infraestrutura de Rede

#### Configurando a Rede

- Conecte seu computador ao switch e configure sua rede para aquela estabelecida no manual do fabricante do Roteador (no caso deste modelo, a rede é 192.168.0.0/24)

- Entre na dashboard do roteador em 192.168.0.1 e, acessando pelas credenciais padrões indicadas no manual, configure a mácara de rede (*Netmask*) para `255.255.240.0` ou `/20`

- Reconfigure seu computador agora para a rede padrão do Switch (no caso deste modelo é 10.0.0.0/8)

- Acesse o Dashboard (10.90.90.90) com as credenciais *default* e reconfigure o Switch com o **IP:** 192.168.0.2/20

#### Configurando a Máquina

Depois de configurar tudo de rede, podemos configurar a NUC:

- Conecte um monitor e um teclado à NUC para realizar a instalação do S.O.

- Instale o Ubuntu Server 20.04 LTS e durante as configurações preencher com as informações abaixo:
    - hostname: nucsoc
    - login: nucsoc
    - senha: socfpga
    - IP Fixo: 192.168.0.3/20

- Verifique se ele consegue *pingar* `8.8.8.8`. Se não conseguir, descubra como rotear os pacotes corretamente.

- Verifique se ele consegue *pingar* `www.google.com`. Se não conseguir, descubra como resolver as urls corretamente.

- Verifique se consegue conectar a ela via **ssh** do seu computador:
```cmd
ssh nucsoc@192.168.0.3
```

- Atualize o **apt**:
```cmd
sudo apt update && sudo apt upgrade -y
```

??? tip 
    Configure a **NAT** no roteador para poder acessar remotamente a NUC dentro da rede do Insper.
    Para isso, configure no roteador:

    - Porta externa: 22
    - Porta interna: 22
    - IP interno: 192.168.0.3

#### Configurando a Placa FPGA

Seguindo os tutoriais dos roteiros da matéria, configure o SDCard com todas as partições necessárias, são elas: **U-Boot** (configurado com a instalação do linux embarcado), **S.O.** e **Filesystem** (buildroot deve estar instalado com todas as dependências de python declaradas mais à frente).

Com tudo configurado, acesse o terminal da NUC via ssh e com o comando de `screen` acesse a Placa conectada a ela.
```cmd
sudo screen /dev/ttyUSB0 115200,cs8
```

Se você configurou NAT, é capaz de acessar a screen da FPGA à distância!

Com esse acesso, agora podemos configurar o ip fixo do S.O. da FPGA:

- Acesse a pasta `/etc/init.d/`

- Crie um arquivo chamado `S60MAC.sh`

!!! info
    Todo o arquivo dentro desta pasta que começa com **S** maiúsculo é executado durante o *boot* do S.O.

- Acessando o arquivo com `vi`, coloque o seguinte código:
```cmd
#!/bin/bash

case "$1" in
start)
    printf "Setting ip: "
    /sbin/ifconfig eth0 192.168.0.40 netmask 255.255.240.0 up
    [ $? = 0 ] && echo "OK" || echo "FAIL"
    route add default gw 192.168.0.1
    ;;
*)  
    exit 1
    ;;
esac
```

O código acima irá configurar a rede da placa para que esta tenha um ip fixo de acordo com as configurações de rede feitas.

- Transforme o arquivo acima em um executável
```cmd
chmod +x S60MAC.sh
```
- Reinicie a FPGA para aplicar as mudanças

**Pronto!** Agora temos toda a inffraestrutura configurada e podemos começar a construir nosso servidor de TFTP.

!!! warning
    Não continue para os próximos passos sem toda a infraestrutura montada e configurada

----------------------------------------------

## Motivação

A nossa motivação para realizar este projeto foi como ele é altamente aplicável a replicação de serviços construídos para SoC-FPGA e no nosso caso, auxiliar a matéria de Embarcados com a possibilidade de automatização dos testes de funcionamento em hardware dos laboratórios.

Além disso, entender sobre os serviços do **GitHub** para automatização de *hooks*, protocolos de transferência de arquivos como o **TFTP**, configuração e boot do DE10-Standard utilizando u-boot e programar no Quartus estavam entre temas de extremo interesse para todo o grupo.