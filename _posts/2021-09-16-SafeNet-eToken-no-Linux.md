---
title: "eToken SafeNet no Linux (Fedora/Debian)"
layout: post
date: 2021-09-16 11:00 -0300
categories: linux segurança token certisign
vertical: Code
excerpt: "Mais um capítulo da eterna briga entre usuários de Linux e usuários do ICP-Brasil com suas ACs que não gostam de suportar Linux."
images:
  - url: /assets/images/icp_brasil.png
    alt: ICP-Brasil
    title: ICP-Brasil
comments: true
---

## Introdução

A briga dos usuários de ceritificados digitais da ICP-Brasil no Linux é eterna e sem perspectiva de fim: A falta de suporte oficial das Autoridades Certificadoras e do próprio governo em seus sistemas são razões para a ínfima adoção de Software Livre nas empresas.

A [Zenith Tecnologia](https://github.com/ZenithTecnologia/) é fortemente baseada em Software Livre e por isso precisa dos sistemas do Governo em estado operacional com Linux. Até o momento tem sido possível seguir sem uso de Softwares Proprietários para o dia a dia da empresa e esta postagem no [blog](https://blog.leonardoamaral.com.br/) busca documentar mais um passo relativo ao uso de sistemas governamentais com Linux: O funcionamento dos SmartCards.

## A escolha da Autoridade Certificadora

Há muitos anos uso a [Certisign](https://www.certisign.com.br/) para emissão de meus certificados [eCPF e eCNPJ](https://www.gov.br/pt-br/servicos/obter-certificacao-digital). A razão para a escolha deles é a organização enquanto empresa para a emissão e validação dos certificados. É fácil marcar a entrevista, nunca tive problemas com os ceritifcados tampouco com os hardwares Criptográficos. No entanto, cada nova compra de SmartCard torna-se uma dor de cabeça e não foi diferente desta vez. 

A Certisign usa soluções da Gemalto (Atualmente comprada pela [Thales](https://www.thalesgroup.com/en)) e normalmente estes SmartCards não funcionam de primeira no Linux. Fui informado ao emitir meu certificado que a tecnologia dos SmartCards havia sido atualizada e fui munido de Notebook para a validação presencial.

Em tempos de pandemia de [SARS-CoV-2](https://www.nature.com/articles/s41591-020-0820-9) - [dispar da ignorância](https://www1.folha.uol.com.br/poder/2021/03/relembre-o-que-bolsonaro-ja-disse-sobre-a-pandemia-de-gripezinha-e-pais-de-maricas-a-frescura-e-mimimi.shtml) - se faz importante ressaltar que é possível a [emissão de certificados validados por Video Conferência](https://www.in.gov.br/en/web/dou/-/instrucao-normativa-iti-n-5-de-22-de-fevereiro-de-2021-304617035). A razão que precisei comparecer ao prédio da Associação Comercial se deu somente devida a necessidade da retirada dos novos cartões presencialmente.

## O novo cartão

O novo cartão me foi descrito como _"Cartão com o chip redondo"_ - [descrição relativamente adequada](https://www.blogs.unicamp.br/dimensional/2009/04/29/044/):

![eCPF e cCNPJ Certisign Frente](/assets/images/certisign_cartao_frente.jpg)

Para quem tiver curiosidade, a parte traseira com a numeração de modelo/lote do cartão (Não é S/N, por isso envio a imagem)

![eCPF/eCNPJ Costas](/assets/images/certisign_cartao_costas.jpg)

A identificação via pcsc_scan é:

```
[root@manaira ~]# pcsc_scan 
Using reader plug'n play mechanism
Scanning present readers...
0: Gemalto PC Twin Reader 00 00
 
Thu Sep 16 12:19:32 2021
 Reader 0: Gemalto PC Twin Reader 00 00
  Event number: 11
  Card state: Card inserted, Shared Mode, 
  ATR: 3B 7F 96 00 00 80 31 80 65 B0 84 56 51 10 12 0F FE 82 90 00

ATR: 3B 7F 96 00 00 80 31 80 65 B0 84 56 51 10 12 0F FE 82 90 00
+ TS = 3B --> Direct Convention
+ T0 = 7F, Y(1): 0111, K: 15 (historical bytes)
  TA(1) = 96 --> Fi=512, Di=32, 16 cycles/ETU
    250000 bits/s at 4 MHz, fMax for Fi = 5 MHz => 312500 bits/s
  TB(1) = 00 --> VPP is not electrically connected
  TC(1) = 00 --> Extra guard time: 0
+ Historical bytes: 80 31 80 65 B0 84 56 51 10 12 0F FE 82 90 00
  Category indicator byte: 80 (compact TLV data object)
    Tag: 3, len: 1 (card service data byte)
      Card service data byte: 80
        - Application selection: by full DF name
        - EF.DIR and EF.ATR access services: by GET RECORD(s) command
        - Card with MF
    Tag: 6, len: 5 (pre-issuing data)
      Data: B0 84 56 51 10
    Tag: 1, len: 2 (country code, ISO 3166-1)
      Country code: 0F FE
    Tag: 8, len: 2 (status indicator)
      SW: 9000

Possibly identified card (using /usr/share/pcsc/smartcard_list.txt):
3B 7F 96 00 00 80 31 80 65 B0 84 56 51 10 12 0F FE 82 90 00
3B 7F .. 00 00 80 31 80 65 B0 .. .. .. .. 12 0F FE 82 90 00
	IDPrime MD 8840, 3840, 3810, 840 and 830 Cards T=0
```

E justamente esta identificação me permitiu achar os modulos corretos.

## Os modulos PKCS11 do cartão

Até aqui, nada demais, a versão mais atual do Safenet Authentication Client é a [10.8](https://data-protection-updates.gemalto.com/2021/07/05/safenet-authentication-client-sac-10-8-for-linux-release-announcement/) com um changelog interessante:

* Rebranding to Thales
* Support for latest version for Ubuntu (v20.04), Red Hat (v8.3), Fedora (v34) and Cent OS (v8.3)
* Support for SafeNet IDPrime 930/3930 and SafeNet IDPrime 3940 FIDO
* Support for GTK3

Sobre o primeiro ítem do changelog, [who cares](https://www.thalesgroup.com/en/group/journalist/press-release/thales-completes-acquisition-gemalto-become-global-leader-digital)? Suportar versões novas de Ubuntu/RHEL/Fedora e CentOS é interessante. O IDPrime novo não faz sentido nesse caso e o Suporte a GTK3 seria muito desejado. [Mas o mundo não é flores](https://gemalto.service-now.com/csm?sys_kb_id=23404c661b4d7c90e2af520f6e4bcbc9&id=kb_article_view&sysparm_rank=1&sysparm_tsqueryId=4f71c86a1b4d7c90e2af520f6e4bcb8e&sysparm_article=KB0024526).

Nos resta a versão 10.7 disponível no link da [GlobalSign](https://www.globalsign.com/): <https://support.globalsign.com/ssl/ssl-certificates-installation/safenet-drivers>.

## Parte genérica da instalação: Pacotes

Não abordarei aqui os procedimentos de descompressão de arquivos pós-descarga. A parte que poucos sabem aqui é como instalar adequadamente pacotes .deb e .rpm (nas versões modernas) e seguem as linhas de exemplo para Fedora e Debian-based:

### Fedora
```
dnf install /home/leonardo/Downloads/Linux\ RPM\ x64/SafenetAuthenticationClient-10.7.77-1.x86_64.rpm
```

### Debian-based
```
apt install /home/leonardo/Downloads/2021_CertisignSmartCard/safenetauthenticationclient_10.7.77_amd64.deb
```

Esta instalação dá acesso imediato ao Safenet Authentication Client Tools:

![SACTools](https://user-images.githubusercontent.com/201189/133641434-36df1825-c089-41ca-9e24-a8c3bca1243b.png)

## Parte não genérica da instalação: Configurações

### Debian

No Debian, por usar em uma máquina com [muitas restrições computacionais](https://gist.githubusercontent.com/leleobhz/e6d849e50da63cf5d86ff1b64c985a5d/raw/c69bcdfa08a69c05f7c8d5cc31b2c0a7e9fc7157/gistfile0.txt), utilizei o certificado via Firefox, bastando adicionar um novo plugin PKCS11 apontado para `/usr/lib/pkcs11/libIDPrimePKCS11.so` 

![Firefox PKCS11](https://user-images.githubusercontent.com/201189/133644220-0d659f27-b53c-4d3b-8405-c1b518b502c3.png)

### Fedora

No Fedora com Chrome - [numa máquina mais modesta ainda](https://gist.githubusercontent.com/leleobhz/2070efd61a54136272647055567a13f4/raw/70a69af508d1afadaba62127d3ef2455386957a0/gistfile0.txt) - a coisa fica diferente por causa do [p11-kit](https://p11-glue.github.io/p11-glue/p11-kit.html)
    
Em geral as pessoas tem medo do diferente, então é comum ouvir reclamações de que ficou mais difícil fazer as coisas. No caso do p11-kit esse caso não se aplica - ao menos não em relação ao [caos que é adicionar módulos PKCS11 no Google Chrome](https://linuxkamarada.com/en/2019/09/26/setting-up-smart-card-authentication-on-google-chrome-chromium/).
    
Bastam dois passos: A criação dos seguintes arquivos (Como root):
    
`/etc/pkcs11/modules/eToken.module`:
    
```
module: libeToken.so
managed: true
```

`/etc/pkcs11/modules/IDPrime.module`
```
module: libIDPrimePKCS11.so
managed: true
```

    E a criação dos links simbólicos corretos:
    
```
ln -s /usr/lib64/libIDPrimePKCS11.so /usr/lib64/pkcs11/
ln -s /usr/lib64/libeToken.so /usr/lib64/pkcs11/
```

E aqui talvez fique explicado o update da versão 10.8 ter citado suporte explicito para distribuições Linux novas: Acredito que estes arquivos já sejam corretamente posicionados. No Fedora, a configuração do p11-kit é suficiente para todos os navegadores (E acredito que até para a autenticação PAM):
    
![Firefox com p11-kit](https://user-images.githubusercontent.com/201189/133646329-2b32fcc7-9c2c-430c-9dd7-6648359e9546.png)

![Chrome com p11-kit](https://user-images.githubusercontent.com/201189/133646831-ec05af8c-7194-4f25-b088-3c924e815d10.png)

### Outras distribuições (?)

Acredito que qualquer distribuição que utilize o p11-kit terá o mesmo procedimento adotado no Fedora - Seja nos pacotes RPM ou nos pacotes DEB<sup id="a1">[1](#f1)</sup>.

## Considerações finais

Pela primeira vez o procedimento foi mais fácil de executar, embora com alguns procedimentos manuais. Não é irrelevante notar que a versão 10.8 resolveria os problemas de acessibilidade em todas as distribuições modernas baseadas em RPM/DEB. Esta observação deveria ser suficiente para cobrar das Autoridades Ceritificadoras que disponibilzem os módulos atualizados de seus sistemas também.

Uma questão que pode ser relevante para escritórios que compartilhem cartões é a presença de um stack de ["cardshare" no p11-kit](https://p11-glue.github.io/p11-glue/p11-kit/manual/remoting.html). Pode ser um recurso muito interessante quando mais de uma pessoa precisa ler o SmartCard sem precisar apelar para métodos de transporte bastante arcaicos quanto o [CPC/CPL](https://vidadeprogramador.com.br/2017/01/30/comunicacao-dpl-dpc/).

<b id="f1">1</b> É preciso notar que na versão deb no lugar da libeToken.so, é instalada na pasta pkcs11 o arquivo `libeTPkcs11.so`. Usualmente a guia de qual arquivo deve ser criado o link simbólico é usar os arquivos contidos em /usr/lib/pkcs11. [↩](#a1)
