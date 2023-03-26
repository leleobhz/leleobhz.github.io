---
title: "Volume in the sky with Docker - Apertem os cintos, o container subiu!"
layout: post
date: 2023-03-25 23:00 -0300
categories: docker container rclone nuvem
vertical: Code
excerpt: "Com a possibilidade do uso de plugins de volume no Docker, o container aprende a voar pras nuvens com o rclone"
images:
  - url: /assets/images/volume_in_sky.jpg
    alt: Volume in the sky
    title: Volume in the sky
comments: true
---

## Introdução 

[O](https://www.docker.com/company/){:target="_blank"} [mundo](https://www.docker.com/blog/inside-look-docker-captains-program/){:target="_blank"} [ama](https://news.itsfoss.com/docker-dropping-free-team-orgs/){:target="_blank"} [Docker](https://www.docker.com/blog/we-apologize-we-did-a-terrible-job-announcing-the-end-of-docker-free-teams/){:target="_blank"}, não é mesmo? Existe por ai uma tonelada de alternativas a ele, inclusive [meu queridinho](https://podman.io/){:target="_blank"}. Eu devo alguns postes sobre o Podman e o ambiente de containers que não seja ambiente dev e também não seja um cluster Kubernetes inteiro e hoje eu venho fazer um pouco de justiça neste ponto.

Muitas aplicações não precisam ser convertidas em micro-serviços e não precisam da distribuição que um ambiente distribuido tem. Ainda não precisando, podem se beneficiar de algumas comodidades dos containers, como consistência de ambiente e configuração, facilidade de operação e facilidade de recovery caso algo exploda em um ambiente de produção.

Hoje apresento mais um motivo pra colocar nesta lista: A possibilidade de gerenciar automaticamente volumes remotos dentro dos containers usando [docker-volume-rclone](https://hub.docker.com/r/rclone/docker-volume-rclone){:target="_blank"}.

## Rclone

O [Rclone é um programa de linha de comando para gerenciar arquivos em armazenamento em nuvem](https://rclone.org/#about:~:text=Rclone%20is%20a%20command%2Dline%20program%20to%20manage%20files%20on%20cloud%20storage.){:target="_blank"}. Esta auto descrição pode levantar suspeitas sobre a necessidade deste post, principalmente quando o leitor descobrir que ele é capaz de [montar volumes](https://rclone.org/commands/rclone_mount/){:target="_blank"} mas não se enganem, ainda será mais prático manter a [sintaxe habitual de volumes do Docker Compose](https://docs.docker.com/compose/compose-file/compose-file-v3/#volumes){:target="_blank"}. Aconselho a todos os aventureios nuvenauticos que se apoderem dessa ferramenta - até mesmo como substituição para [rsync](https://rclone.org/commands/rclone_sync/){:target="_blank"}/[sftp](https://rclone.org/sftp/){:target="_blank"}/[ftp](https://rclone.org/ftp/){:target="_blank"}/[mount](https://rclone.org/commands/rclone_mount/){:target="_blank"}.[cifs](https://rclone.org/smb/){:target="_blank"} e tantas outras ferramentas que temos por ai.

Aos que ainda não se convenceram do poder do Rclone, há duas ferramentas do Rclone - [hasher](https://rclone.org/hasher/){:target="_blank"} e [compress](https://rclone.org/compress/){:target="_blank"} - que podem ser aninhadas em qualquer outro endpoint (E aninhadas entre si também - como faremos adiante), adicionando algum poder a protocolos mais restritos, como o [ftp](https://rclone.org/ftp/){:target="_blank"}.

## Docker plugin

O Docker tem uma plataforma de [plugins](https://docs.docker.com/engine/extend/){:target="_blank"} bem mais interessante que os [plugins do podman](https://github.com/containers/common/blob/main/docs/containers.conf.5.md#:~:text=A%20table%20of%20all%20the%20enabled%20volume%20plugins%20on%20the%20system.%20Volume%20plugins%20can%20be%20used%20as%20the%20backend%20for%20Podman%20named%20volumes.%20Individual%20plugins%20are%20specified%20below%2C%20as%20a%20map%20of%20the%20plugin%20name%20(what%20the%20plugin%20will%20be%20called)%20to%20its%20path%20(filepath%20of%20the%20plugin%27s%20unix%20socket).){:target="_blank"} em termos de facilidade de deploy de novas expansões. Não avaliei se o podman é capaz de rodar a mesma estrutura da [plugin API do Docker](https://docs.docker.com/engine/extend/plugin_api/){:target="_blank"} - deixando a sugestão ao leitor a possibilidade de buscar esta posibilidade - e quem sabe a publicar também. Esta plataforma de plugins será utilizada para __mapear em volumes Docker qualquer serviço suportado pelo Rclone__.

## Bora passar raiva? Booooooooraa!

### Pré-requisitos

Para o procedimento, o paacote fuse se faz necessário. Nas duas bases mais comuns, o processo de instalação costuma ser o mesmo:

* `apt install fuse`
* `yum install fuse`
* `dnf install fuse`

Também é necessária a criação de duas pastas para armazenar o cache e a configuração [^1]:

```bash
sudo mkdir -p /var/lib/docker-plugins/rclone/config
sudo mkdir -p /var/lib/docker-plugins/rclone/cache
```

### Instalação do plugin

Atendidos os [pré-requisitos](#pré-requisitos), hora de instalar o plugin!

```bash
docker plugin install rclone/docker-volume-rclone:amd64 \
args="-v" --alias rclone --grant-all-permissions
```

E ver se tudo está instalado como deveria:

```bash
docker plugin list
```

É esperado um resultado como o seguinte:

```
$ docker plugin list
ID             NAME            DESCRIPTION                       ENABLED
c0ef4f512e4f   rclone:latest   Rclone volume plugin for Docker   true
```

## Voa passarinho, voa

Vou deixar propositalmente [os dois passos que já estão documentados pela equipe do Rclone](https://rclone.org/docker/){:target="_blank"} de fora deste texto. O cenário mais simples que cria manualmente o volume pode ser compreendido partindo da leitura deste texto - mesmo sem conhecimento da língua inglesa. O cenário envolvendo _docker-compose_ está ilustrado, mas devo trazer uma visão um pouco mais prática de como chegar a um ponto de uso útil da ferramenta - ilustrando um backup simples de _MySQL_ como exemplo.

### Configurando os serviços

A pasta onde o Rclone instala seu arquivo de config é a `/var/lib/docker-plugins/rclone/config/` e a configuração do Rclone é gerada pelo comando [`rclone config`](https://rclone.org/commands/rclone_config/). Juntando as duas coisas com um container pra configurar o Rclone (Afinal, pra que motivo iremos instalar via gerenciador de pacotes?), fica da seguinte forma:

```bash
# Container para configurar o Rclone

docker run -it --rm --name rclone_config -v /var/lib/docker-plugins/rclone/config/:/config/rclone rclone/rclone config
```

Nesta altura do campeonato, já fica posto que é possível usar variações do comando acima para fazer qualquer operação fora dos containers envolvendo os serviços configurados para os volumes do Docker, como por exemplo:

```bash
# Vamos listar os arquivos do serviço brasil_bagunca:
# E não podemos esquecer do dois-pontos no final!

docker run -it --rm --name rclone_config -v /var/lib/docker-plugins/rclone/config/:/config/rclone rclone/rclone ls brasilbagunca:
```

E assim deverá ser criada toda as configurações para que o Rclone possa operar e vou deixar um exemplo que demonstra um dos poderes interessantes do Rclone:

{% highlight init linenos %}
[brasilbagunca]
type = ftp
host = senhoresselva.com.br
user = cristiano
pass = seu_lixo
explicit_tls = true

[brasilbagunca_compress]
type = compress
remote = brasilbagunca:

[brasilbagunca_compress_hash]
type = hasher
remote = brasilbagunca_compress:
{% endhighlight %}

Criamos um perfil de acesso chamado _brasilbagunça_ que se conecta em um FTP. Como não tem como fazer hash e compressão no FTP, criamos um novo perfil de acesso chamado _brasilbagunca_compress_ que comprime os arquivos em gzip e envia para o remote _brasilbagunca_.

__AVISO: O Rclone diz [explicitamente](https://rclone.org/compress/#:~:text=The%20file%20names%20should%20not%20be%20changed%20by%20anything%20other%20than%20the%20rclone%20compression%20backend.) que é possível acessar os arquivos e que modifica-los é perigoso! Não altere os arquivos gerados no destino! No lugar disso, veja acima o exemplo que usamos com _rclone ls_.__

E como FTP também não suporta hash remoto, adotamos o componente _hasher_ em cima do componente _compress_. O _hasher_ gera arquivos .json para cada arquivo enviado com o hash local e verifica este hash em funções como `copy`, `sync` e etc. 

No final das contas, com uma configuração destas, é possível enviar dados para o FTP com um hash e compressão dos arquivos, o que facilita bastante a vida ao utilizar sistemas legados. Como a configuração é aninhada, usaremos a configuração que contempla os três sistemas, como no exemplo abaixo:

```bash
# Vamos listar os arquivos do serviço brasil_bagunca após a compressão e o hashing:
# E novamente não podemos esquecer do dois-pontos no final!

docker run -it --rm --name rclone_config -v /var/lib/docker-plugins/rclone/config/:/config/rclone rclone/rclone ls brasilbagunca_compress_hash:
```

### Forjando um _docker-compose.yml_

E chega a hora de juntar isso em uma configuração útil. Para manter os serviços o mais modular possível, utilizei o agendador [Ofelia](https://github.com/mcuadros/ofelia) - um sistema construído em Go que permite utilizar labels do Docker para parametrizar o agendador. O container [MariaDB](https://hub.docker.com/_/mariadb) é padrão e só tem a mais uma pasta de backup que será mapeada para o volume criado com o driver _rclone_. As rotinas e configurações do _job-exec_ podem ser consultadas [aqui](https://github.com/mcuadros/ofelia/blob/master/docs/jobs.md). Para os entendidos de _cron_ e _mysql_, os comandos serão intuitovos e não explicados neste artigo para não desviar demais.

{% highlight yaml linenos %}
version: '3.8'
name: fora_do_brasil

services:
  ofelia:
    image: mcuadros/ofelia:latest
    command: daemon --docker
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  mariadb:
    image: mariadb:latest
    restart: unless-stopped
    labels:
      ofelia.enabled: true
      ofelia.job-exec.mysql-optimize-fora_do_brasil.schedule: "@daily"
      ofelia.job-exec.mysql-optimize-fora_do_brasil.command: "mysqlcheck -uroot -ppiano_caindo -o --all-databases"
      ofelia.job-exec.mysql-backup-fora_do_brasil.schedule: "@daily"
      ofelia.job-exec.mysql-backup-fora_do_brasil.command: 'bash -c "mysqldump -uroot -ppiano_caindo --all-databases >> /backup/$(date +%Y%m%d-%H%M%S).sql"'
      ofelia.job-exec.mysql-clean-fora_do_brasil.schedule: "@daily"
      ofelia.job-exec.mysql-clean-fora_do_brasil.command: 'bash -c "find /backup -type f -mtime +7 -print0 | xargs -r0 rm -v --"'
    environment:
      - MYSQL_ROOT_PASSWORD=piano_caindo
    volumes:
      - backup:/backup

volumes:
  backup:
    driver: rclone
    driver_opts:
      remote: 'brasilbagunca_compress_hash:vejam/voces/percebem/a/loucura/'
      allow_other: 'true'
      vfs_cache_mode: full
      poll_interval: 0

{% endhighlight %}

Pode chamar atenção do leitor mais cuidadoso a opção _allow_other_ que é mandatória: Esta opção permite que mais de um volume no mesmo remote seja lançado. Caso esta opção não seja utilizada, somente um volume será montado para cada remote. 

A opção _vfs_cache_mode_ está explicada [neste link](https://rclone.org/commands/rclone_mount/#vfs-file-caching) e resumidamente se comporta da seguinte forma: É preciso escolher um modo de cache para qualquer operação que não seja sequencial e que contemple as limitações que cada uma das opções de cache oferece (_off_, _minimal_, _writes_ e _full_). O uso da opção _full_ considerando que é um cenário de backup é adequada, porém cada cenário poderá exigir uma configuração diferente.

A opção _poll_interval_ determina o tempo que o Rclone irá atualizar a lista local com a lista remota. No caso deixei em 0 porque é fundamentalmente um sistema de somente escrita, então não vejo razão para carregar o remoto com polling. Entretanto - conforme sinalizado em outros casos - a situação pode demandar mudanças.

Recomendo a leitura da página de informações sobre o [rclone mount](https://rclone.org/commands/rclone_mount) pois as opções desta página se aplicam para a montagem dos volumes também.

## Conclusão

Este artigo visa mostrar o poder dos plugins do Docker e também indicar o poderoso _Rclone_. Os dois juntos podem ser uma excelente ferramenta para quem está convertendo estruturas on premisse sem containers para containers em qualquer formato - sem necessariamente mudar os paradígmas da solução existente. 

Caso tenha dúvidas, sugestões ou alguma observação, dá um toque nos comentários abaixo ou nas redes sociais no menu acima!

E nunca se esqueça, se for [fultonar](https://metalgear.fandom.com/wiki/Fulton_surface-to-air_recovery_system) um container, segure-se bem e aproveite a viagem! 

![Metal Gear Auto Extração via Fulton](/assets/images/mgs_fulton.jpg)

[^1]: O comando `sudo` será indicativo - salvo por outra necessidade - de que há necessidade de execução como root. Poderia indicar usando `#` e `$`, porém acredito que esta forma é mais verbosa e compreensível. 
