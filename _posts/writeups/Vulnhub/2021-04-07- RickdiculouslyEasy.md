---
title: Vulnhub - RickdiculouslyEasy
categories: [Vulnhub]
#tags: [BurpSuite,Binwalk,Bruteforce]
---

![](/assets/img/Vulnhub/RickdiculouslyEasy/capa.png)

Link da Materia: <https://www.vulnhub.com/entry/rickdiculouslyeasy-1,207/>


# Scan/Enumeracao


## Host Discovery


* Com o arp-scan descobrimos que o nosso alvo esta no ip **192.168.150.118**

```bash
sudo arp-scan -I eth1 192.168.56.0/24
```

![](/assets/img/Vulnhub/RickdiculouslyEasy/arp-scan.png)


## Port Discovery


* Com o parametro "-p-" do nmap varremos todas as portas possiveis do nosso alvo e encontramos 07 delas abertas. **21**, **22**, **80**, **9090**, **13337**, **22222** e **60000**

```bash
sudo nmap -n -T5 -p- 192.168.56.105
```

![](/assets/img/Vulnhub/RickdiculouslyEasy/port-discovery.png)



## Port Scan


* Agora com o parametro "-A" do nmap vamos descobrir mais informacoes sobre essas portas abertas do nosso alvo. Temos algumas portas que retornaram o "**tcpwrapped**", isso significa que o nmap identificou algum servico analogo ao "TCP Wrapped" que limita o acesso a algumas aplicacoes a host previamente configurados. Ou seja, como nao estamos na lista dos que podem acessar a aplicacao o Handshake e concluido porem o servidor encerra a conexao, por isso a porta aparece como tcpwrapped...

```bash
sudo nmap -n -T5 -A -p 21,22,80,9090,13337,22222,60000 192.168.56.105
```

![](/assets/img/Vulnhub/RickdiculouslyEasy/port-scan.png)

> - Porta 21: vsftpd 3.0.3
> - Porta 22: tcpwrapped
> - Porta 80: Apache httpd 2.4.27 ((Fedora))
> - Porta 9090: Cockpit web service 161 or earlier
> - Porta 13337: tcpwrapped
> - Porta 22222: OpenSSH 7.5 (protocol 2.0)
> - Porta 60000: tcpwrapped


# Enumeracao/Exploracao Web (Porta 80)


## Browser pagina principal

* Entramos na pagina da aplicacao Web

![](/assets/img/Vulnhub/RickdiculouslyEasy/web-80.png)



## Fuzzy de diretorios - Gobuster

* Como e de costume, vamos rodar o gobuster em cima da pagina para ver o que encontramos de interessante... Achamos o "**/cgi-bin/**", "**/passwords**" e o "**/robots.txt**"

![](/assets/img/Vulnhub/RickdiculouslyEasy/gobuster-web80.png)


### /passwords

* Acessando o diretorio "passwords" temos dois arquivos "**flag.txt**" e "**passwords.html**"

![](/assets/img/Vulnhub/RickdiculouslyEasy/passwords-flag.png)

![](/assets/img/Vulnhub/RickdiculouslyEasy/passwords-passwords-html.png)



* No arquivo "passwords.html" esta falando que "Morty" deixou a senha em um lugar facil, entao o outro usuario disse que vai esconder melhor... Olhando o codigo fonte achamos algo interessante "**Password: winter**" 

![](/assets/img/Vulnhub/RickdiculouslyEasy/passwords-codigoFonte-html.png)


* Tentamos ssh com o usuario "morty" e a senha encontrada, porem, sem sucesso...


### /robots.txt

* Acessando o robots.txt temos algumas paginas interessantes: "**root_shell.cgi**" e "**tracertool.cgi**"

![](/assets/img/Vulnhub/RickdiculouslyEasy/robots-txt.png)


#### root_shell.cgi

* Entramos na pagina porem ela esta em contrucao...

![](/assets/img/Vulnhub/RickdiculouslyEasy/root-shell-cgi.png)


* Olhando o codigo fonte nao temos nada de interessante, apenas alguns comentarios... Coisa de CTF

![](/assets/img/Vulnhub/RickdiculouslyEasy/root-shell-cgi-codigoFonte.png)



#### tracertool.cgi

* A pagina pede para informarmos um IP... aparentemente ele ira nos tetornar a rota ate o IP

![](/assets/img/Vulnhub/RickdiculouslyEasy/tracertool-cgi.png)


* Testamos e a pagina nos retornou algo semelhante com o comando traceroute 

![](/assets/img/Vulnhub/RickdiculouslyEasy/tracertool-cgi-I.png)


## BurpSuite

* Vamos para o Burp, para deixar mais perfermatica a nossa enumeracao

* Analisando a requisicao vemos como a pagina passa o IP

![](/assets/img/Vulnhub/RickdiculouslyEasy/burp-tracertool.png)



* Agora vamos jogar para o "repeater" do Burp para ver se conseguimos executar comando nessa pagina... Conseguimos RCE!

![](/assets/img/Vulnhub/RickdiculouslyEasy/burp-rce.png)


* Depois de varias tentativas tentando executar algum comando para nos retornar o shell nao obtivemos sucesso em nenhuma... O que nos resta e dar uma olhada nos arquivos na maquina para tentar achar alguma vulnerabilidade...


## Usuarios

* Vamos dar uma olhada nos usuarios da maquina, lendo o arquivo /etc/passwd, ja que nos tivemos uma dica de senha de um deles...

* Parece que a aplicacao ta filtrando o comando "cat", para contornar isso conseguimos ler o passwd usando o comando "**tail**". Achamos 03 ususarios que podemos realizar login na maquina: "**RickSanchez**", "**Morty**" e "**Summer**"

```bash
192.168.56.101;tail -f /etc/passwd
```

![](/assets/img/Vulnhub/RickdiculouslyEasy/passwd-tracetool.png)


# SSH - Porta 22222

* Antes de tudo vamos tentar logar com um desses usuarios que acabamos de encontrar e a senha que achamos no codigo fonte de uma das paginas. "**winter**"

* Conseguimos logar com a credencial **Summer:winter**

![](/assets/img/Vulnhub/RickdiculouslyEasy/ssh-summer.png)


# Usuario Summer 

* Achamos mais uma flag

![](/assets/img/Vulnhub/RickdiculouslyEasy/flag-summer.png)


* Vamos dar uma olhada nos arquivos da maquina... Conseguimos entrar no **/home/Morty**. Tem dois arquivos la, vamos transferi-los para a nossa maquina

![](/assets/img/Vulnhub/RickdiculouslyEasy/home-morty.png)

![](/assets/img/Vulnhub/RickdiculouslyEasy/scp-home-morty.png)


* Conseguimos transferir os arquivos para a nossa maquina, mas nao conseguimos descompactar o "journal.txt.zip" e o "Safe_Password.jpg" aparentemente nao tem nada...

![](/assets/img/Vulnhub/RickdiculouslyEasy/arquivos-home-morty.png)


* No diretorio "**/home/RickSanchez**" temos mais dois diretorios, o **RICKS_SAFE** que tem um binario dentro dele, porem nao temos permissao de execucao e o **ThisDoesntContainAnyFlags** que tem um arquivo .txt (nao e uma flag)

![](/assets/img/Vulnhub/RickdiculouslyEasy/home-rick.png)


* Depois de varias tentativas conseguimos algo interessante... No arquivo **Safe_Password.jpg**, que esta no /home/Morty, contem uma senha... Basta verificar as strings do arquivo que conseguiremos ver essa dica

```bash
strings Safe_Password.jpg
```

![](/assets/img/Vulnhub/RickdiculouslyEasy/strings-Safe-Password.png)


* Exitem outros jeitos de descobrir esse segredo do arquivo .jpg, uma dela e dar um "cat" no arquivo...

![](/assets/img/Vulnhub/RickdiculouslyEasy/cat-Safe-Password.png)


* A melhor maneira que encontrei para verificar essa informacao foi com o comando "**binwalk**". Encontramos a senha **Meeseek** para descompactar o arquivo .zip

```bash
binwalk -e Safe_Password.jpg
```

![](/assets/img/Vulnhub/RickdiculouslyEasy/binwalk-Safe-Password.png)


* Descompactando o arquivo "journal.txt.zip" temos um recado dizendo algo sobre uma senha de seguranca: **131333**

![](/assets/img/Vulnhub/RickdiculouslyEasy/journal-txt.png)

> - Bom, temos mais uma flag porem nao saimos do lugar ate agora...


* Vamos mover o binario "safe" para o /tmp, dar permissao e executa-lo...

![](/assets/img/Vulnhub/RickdiculouslyEasy/safe.png)

> - O binario diz que temos que passar argumentos



* Depois de varias tentativas e o programa retornando nada de interessante, passamos a senha de seguranca que encontramos. 

![](/assets/img/Vulnhub/RickdiculouslyEasy/safe-131333.png)

> - A dica e para criarmos um script para gerar senhas com as seguinte dicas:
>   - 01 caracter maiusculo
>   - 01 numero (digito)
>   - 01 das palavras da "minha antiga banda"


### Criando uma wordlist

* Primeiramente vamos procurar qual era o nome da banda...

![](/assets/img/Vulnhub/RickdiculouslyEasy/banda-rick.png)


* Agora vamos criar um script em python para gerar uma wordlist de acordo com as dicas que achamos. 

```python
#!/usr/bin/python3
import string
for letra in list(string.ascii_uppercase):
    for numero in [0,1,2,3,4,5,6,7,8,9]:
        for palavra in ['The', 'Flesh', 'Curtains']:
            print(str(letra)+str(numero)+str(palavra))
```

![](/assets/img/Vulnhub/RickdiculouslyEasy/gerador-wordlist-I.png)

* Agora e so dar permissao de execucao para o script e executa-lo jogando a sua saida para um arquivo

![](/assets/img/Vulnhub/RickdiculouslyEasy/wordlist.png)


# Brute Force SSH - Porta 22222

* Bom, agora que ja temos uma wordlist vamos tentar quebrar a senha do ssh. Como nao tenho certeza do ususario eu vou tentar fazer com o dois que conhecemos na maquina... Conseguimos achar a credencial **RickSanchez:P7Curtains**

```bash
hydra -L user.txt -P wordlist_II.txt ssh://192.168.56.105 -s 22222
```

![](/assets/img/Vulnhub/RickdiculouslyEasy/credencial-rick.png)



# Usuario RickSanchez

* Conseguimos logar com a credencial encontrada

![](/assets/img/Vulnhub/RickdiculouslyEasy/shell-rick.png)


* Demos sorte na primeira tentariva... Verificamos que o usuario Rick tem permissao para fazer qualquer coisa com o sudo na maquina 

```bash
sudo -l
```

![](/assets/img/Vulnhub/RickdiculouslyEasy/sudo-rick.png)


* Agora ficou facil, e so logar com o root usando o sudo 

```bash
sudo su
```

![](/assets/img/Vulnhub/RickdiculouslyEasy/shell-root.png)


* **Pronto, agora somos ROOT!**


# ROOT

* Achamos mais uma flag, a do **ROOT**

![](/assets/img/Vulnhub/RickdiculouslyEasy/flag-root.png)


# Bonus

## Flag Porta 13337

* Conseguimos pegar mais uma flag que esfava na porta 13337 da maquina

![](/assets/img/Vulnhub/RickdiculouslyEasy/flag-13337.png)


## FTP - Porta 21

* Conseguimos acessar o ftp como anonymous e achamos mais uma flag

![](/assets/img/Vulnhub/RickdiculouslyEasy/flag-ftp.png)



## Web - Porta 9090

* Entrando na pagina web da porta 9090 temos mais uma flag e somente isso... Nao achamos nada de interessante...

![](/assets/img/Vulnhub/RickdiculouslyEasy/flag-9090.png)


# Consideracoes Finais

* Seguindo uma ordem de prioridade conseguimos explorar a maquina sem perder muito tempo com os servicos rolhas. Talves teria como acessar a maquina de outra maneira, mas por hora nao vamos explorar os outros servicos.