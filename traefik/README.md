# Traefik central da máquina

Esta pasta fica no repositório `home-lab-docker` e contém o proxy compartilhado
entre os projetos da máquina. Cada aplicação mantém seu próprio Compose.
Requer Docker com containers Linux e Compose v2.

## Subir

Na raiz do repositório `home-lab-docker`:

```bash
cd traefik
docker compose up -d --wait
docker compose ps
```

O comando cria o container `server-traefik` e a rede `traefik-proxy`.
O proxy atende em `http://127.0.0.1:8090`, disponível para o Cloudflare Tunnel
que roda na mesma máquina. A porta pode ser alterada criando um `.env` a partir
de `.env.example`. HTTPS externo continua sendo atendido pelo Cloudflare;
esta configuração não solicita certificados nem abre portas 80/443 no host.

Um `404` ao acessar somente `http://localhost:8090` é esperado: o encaminhamento
depende do domínio declarado nas labels. O dashboard não está habilitado.

## Conferir a descoberta automática

Com o proxy rodando, na mesma pasta:

```bash
docker compose -f examples/whoami/compose.yaml up -d
curl -i -H 'Host: traefik.localhost' http://127.0.0.1:8090
```

No PowerShell, use `curl.exe` nesse comando. A resposta deve ser HTTP 200,
com informações da aplicação de exemplo. O exemplo não publica uma porta própria:
o acesso passa pelo Traefik. Após conferir:

```bash
docker compose -f examples/whoami/compose.yaml down
```

## Conectar outros projetos

Cada Compose de aplicação conecta seu serviço web à rede externa
`traefik-proxy` e declara domínio e porta interna nas labels. Consulte
`examples/whoami/compose.yaml`; substitua imagem, domínio, porta e o nome
`proxy-example` por valores exclusivos do projeto.

As demais redes privadas do projeto, por exemplo para banco, podem continuar.
Somente o serviço web precisa entrar na rede do proxy. Não coloque labels de
publicação no banco ou no RabbitMQ. A descoberta usa o Docker compartilhado da
máquina; Docker rootless separado por usuário exige outra configuração.

O Traefik precisa consultar a API do Docker e recebe o socket do daemon.
A montagem `:ro` não transforma essa API em uma API com permissões limitadas;
esta base pressupõe projetos e administradores confiáveis no mesmo daemon.

## Cloudflare Tunnel

Quando uma aplicação já estiver conectada e testada pelas labels, o serviço
de seu hostname no Tunnel poderá apontar para:

```text
http://localhost:8090
```

O hostname original deve ser preservado, pois a regra `Host(...)` o utiliza.
DNS e o hostname do Tunnel são configurados separadamente das labels.
Esta base pressupõe `cloudflared` instalado no host; dentro de outro container,
`localhost` não aponta para o Traefik.

O OpenJobs atual usa `network_mode: host` e a API escuta em localhost. O exemplo
com rede bridge não pode ser aplicado diretamente nessa API: sua conexão ao
proxy será uma etapa separada. Subir esta base não muda o destino do OpenJobs.

## Comandos úteis

```bash
docker compose logs --tail=100 traefik
docker compose restart traefik
docker compose config --quiet
```

Alterações nas labels são detectadas quando o container da aplicação é criado
ou recriado. Alterações em `traefik.yaml` exigem reiniciar o Traefik.

Fontes:
- https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/observability/healthcheck/
