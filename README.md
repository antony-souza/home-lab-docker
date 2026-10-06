# Home Lab Docker

Infraestrutura Docker compartilhada pelos projetos do servidor.

## Traefik

O diretório `traefik/` contém o Compose do proxy, sua configuração e um exemplo
de aplicação com labels. Ele cria a rede compartilhada `traefik-proxy` e atende
na porta local `8090`, para uso pelo Cloudflare Tunnel instalado no host.

### Subir no servidor

```bash
git clone https://github.com/antony-souza/home-lab-docker.git
cd home-lab-docker/traefik
docker compose up -d --wait
docker compose ps
```

Depois de publicar alterações neste repositório:

```bash
git pull --ff-only
docker compose up -d --wait
```

Se alterar somente `traefik.yaml`, execute também `docker compose restart traefik`,
pois um arquivo montado não força a recriação do container.

As aplicações se conectam à rede externa `traefik-proxy` e informam domínio e
porta interna pelas labels. Não é necessário editar a configuração central para
registrar cada projeto. O DNS e o hostname no Cloudflare Tunnel são configurados
separadamente.

Consulte [as instruções completas](traefik/README.md) e
[o exemplo de aplicação](traefik/examples/whoami/compose.yaml).

## Cloudflare Tunnel por usuário

O diretório `cloudflared/` contém um Compose para executar um conector de Tunnel
em container. Cada usuário usa o token de um Tunnel criado na própria conta
Cloudflare e um nome de projeto exclusivo. Todos podem encaminhar para o Traefik
central em `http://server-traefik:80`, pela rede compartilhada `traefik-proxy`.

Consulte [as instruções do conector](cloudflared/README.md) para configurar o
domínio no painel, preencher o `.env`, subir e testar o container.
