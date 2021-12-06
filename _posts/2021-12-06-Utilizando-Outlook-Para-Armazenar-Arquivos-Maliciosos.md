---
layout: single
classes: wide
title:  "Utilizando o Outlook para Armazenar Arquivos Maliciosos"
---
## Descrição
Conseguimos através do anexo de e-mail do Outlook armazenar arquivos maliciosos.
Isso nos permite realizar o upload/download de arquivos maliciosos através de um “local confiavel” isso por se tratar de um subdomínio do Outlook.


## Passos para reproduzir
Primeiramente devemos modificar nosso arquivo malicioso adicionando uma extensão de texto (.txt), se o ambiente em que queremos realizar o download do arquivo malicioso é um Windows muito provavelmente o arquivo será um executavel (.exe).
No Linux podemos incluir essa extensão de várias formas, uma delas é utilizar o comando *cp* para copiar o arquivo e cria outro arquivo adicionando a extensão que desejarmos, nesse caso ficaria algo como: *arquivo.exe.txt*
Após prepararmos o arquivo malicioso podemos abrir o Outlook e irmos na opção “Nova Mensagem” e nela anexar nosso arquivo modificado e enviar para um destino qualquer pois o que nos interessa é o link do arquivo para download.
Depois de enviarmos podemos clicar para baixar o arquivo e assim obter o link direto do arquivo clicando na aba Downloads do Navegador > Botão direito >  Copiar link.


## Prova de Conceito (PoC)



## Observações
No GIF acima realizamos o download diretamente no “ambiente da vítima” isso para termos um acesso mais rápido ao link de download. Porém em um cenário real deveríamos escolher outro meio de compartilhar esse link com a vítima para que fosse feito o Download do arquivo malicioso.
Recomendamos não realizar o upload de arquivos apenas em .exe pois o outlook detecta como arquivo malicioso e impede que seja realizado o download, também arquivos muito grandes são bloqueados por conta da restrição de tamanho do arquivo por parte do anexo.
Um outro ponto interessante é que o arquivo após enviado tem um prazo de validade que vária entre 10 à 15 minutos

## Cénario de Ataque
Este tipo de ataque permite que um invasor execute comandos maliciosos no Google Assistente, em nome da vítima, apenas fazendo a vítima visitar uma página da web.
O Assistente tem acesso a informações extremamente confidenciais e pode ser capaz de controlar a Conta do Google da vítima, o Smart Home e outros dispositivos IOT.
Um invasor nunca deve ser capaz de executar comandos no Google Assistente sem ter permissões suficientes.
