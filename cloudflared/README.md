# Cloudflare Tunnel por usuário

Este Compose executa um `cloudflared` para um Tunnel gerenciado pelo painel
Cloudflare. Cada usuário cria seu Tunnel na própria conta e usa seu próprio token.
Os conectores encaminham para o mesmo Traefik central pela rede `traefik-proxy`.

```text
Domínio do usuário → Cloudflare → Tunnel do usuário → Traefik → aplicação
```

## Configurar no Cloudflare

1. Crie um Cloudflare Tunnel na conta que contém seu domínio.
2. Selecione a instalação com Docker e copie apenas o token do comando mostrado
   pelo painel. O token é uma credencial; não o coloque no Git.
3. Adicione uma rota de aplicação publicada para seu domínio, por exemplo
   `api.example.com`, com serviço do tipo HTTP e destino `server-traefik:80`.
   O endereço completo é `http://server-traefik:80`.
4. Mantenha o Host original da requisição; não configure uma substituição do
   HTTP Host Header. O Traefik usa esse domínio para escolher a aplicação.
5. Confira o registro DNS criado pelo painel. Se precisar criá-lo manualmente,
   use um CNAME com proxy ativado apontando para `<ID_DO_SEU_TUNNEL>.cfargotunnel.com`.

Use o ID do seu próprio Tunnel, não o de outro usuário. O HTTPS público é
atendido pelo Cloudflare; a conexão interna ao Traefik usa HTTP.

## Subir no servidor

O Traefik central precisa estar rodando e ter criado a rede `traefik-proxy`.
Este exemplo usa o mesmo Docker compartilhado do Traefik, não um Docker rootless
independente por usuário.

Na sua cópia do repositório:

```bash
cd cloudflared
cp .env.example .env
chmod 600 .env
nano .env
```

Configure `TUNNEL_NAME` com um nome exclusivo, como `tunnel-walker`, e preencha
`TUNNEL_TOKEN` com a credencial copiada do painel. Cada usuário usa sua própria
cópia e seu próprio `.env`. O Compose não fixa `container_name`, permitindo que
vários projetos de Tunnel rodem juntos sem conflito de nomes.

```bash
docker compose config --quiet
docker compose up -d
docker compose ps
docker compose logs --tail=50 tunnel
```

No painel, confira se o Tunnel aparece saudável. Nos logs, procure conexões
registradas com o Cloudflare. O container não publica portas no host e não
precisa montar o socket Docker.

## Publicar e testar uma aplicação

Conecte a aplicação à rede `traefik-proxy` e configure suas labels do Traefik com
o mesmo domínio cadastrado no painel e a porta interna correta. Use nomes
exclusivos de router e service. Consulte o
[exemplo de labels](../traefik/examples/whoami/compose.yaml).

```bash
curl -i -H 'Host: api.example.com' http://127.0.0.1:8090/
curl -i https://api.example.com/
```

Substitua o domínio e o caminho por um endpoint existente na sua aplicação.
O primeiro comando verifica o Traefik local; o segundo inclui o DNS e o Tunnel.
Dentro do container, `localhost` aponta para o próprio container: use
`server-traefik:80` como destino do Tunnel.

## Atualizar ou parar seu conector

```bash
docker compose pull
docker compose up -d
```

Para parar somente o projeto de Tunnel desta pasta:

```bash
docker compose down
```

Cada token pode iniciar réplicas do mesmo Tunnel. Use tokens de Tunnels diferentes
quando quiser conectores de contas diferentes. Não é necessário alterar ou parar
o conector já instalado como serviço no servidor.

Este modelo compartilha o Docker e a rede entre os projetos; não fornece
isolamento por usuário nem reserva domínios para um usuário específico.

Referência: [configuração oficial do Cloudflare Tunnel](https://developers.cloudflare.com/tunnel/get-started/).
