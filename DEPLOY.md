# Deploy do site AI4ALL — Runbook

> Última atualização: 02/07/2026
> Objetivo: subir um fix do site em poucos minutos, sem redescobrir a infra toda a cada vez.

---

## 1. Arquitetura em uma tela

| Item | Valor |
|---|---|
| Repositório | `https://github.com/Tams-1/ai4all-website` (branch `main`) |
| Pasta na VPS | `/docker/ai4all-website` (clone do repo) |
| Build | `Dockerfile` na raiz → imagem `nginx:alpine` que copia os arquivos estáticos |
| Imagem | `docker-ai4all-website` |
| Container | `ai4all-website` (porta 80 interna, **não publicada**) |
| Rede | `bridge` (rede default do Docker) |
| Restart policy | `unless-stopped` |
| Roteamento | **Traefik** (container `traefik-traefik-1`), via labels no container |
| Domínios | `ai4allplatform.com`, `www.ai4allplatform.com`, `ai4allplatform.com.br`, `www.ai4allplatform.com.br` |
| TLS | Let's Encrypt (`certresolver=letsencrypt`), entrypoint `websecure` |

**Importante:** existem 3 coisas distintas, não confundir:
1. **AI4ALL Platform** → `/docker/ai4all-platform` (repo `Tams-1/AI4ALL-Platform`)
2. **AI4ALL Dashboard** → `/docker/ai4all-dashboard`
3. **AI4ALL Website** → `/docker/ai4all-website` (repo `Tams-1/ai4all-website`) ← **este runbook é sobre o item 3**

---

## 2. Arquivos que vivem SÓ na VPS (não estão no repo)

Na pasta `/docker/ai4all-website` existem arquivos **untracked** (não commitados) que são necessários para o build e **não podem ser apagados**:

- `Dockerfile` — define o build (nginx:alpine + copy dos estáticos).
- `nginx.conf` — config do nginx (`root /usr/share/nginx/html`, `try_files $uri $uri/ $uri.html =404`).

Além disso, o `index.html` na VPS tem uma alteração local **não commitada**: a meta tag de verificação de domínio do Facebook logo após o `<head>`:

```html
<meta name="facebook-domain-verification" content="e3jzix0pwxq0pixou03855dt0yfk6g" />
```

Por isso, **todo `git pull` na VPS precisa preservar essa alteração local** (ver passo a passo abaixo). Se ela se perder, a verificação do Facebook quebra.

> **TODO futuro:** commitar `Dockerfile`, `nginx.conf` e a meta tag do Facebook no repositório, para eliminar o "stash dance" e o risco de perder arquivos.

---

## 3. Deploy de um fix — passo a passo

### 3.1. No seu computador (edição + push)

```bash
cd "C:\Users\tamsi\Documents\AI4ALL\ai4all-website"
# editar os arquivos (ex.: index.html, termos.html, privacidade.html)
git add -A
git commit -m "descrição do fix"
git push origin main
```

### 3.2. Na VPS (pull preservando alterações locais)

```bash
cd /docker/ai4all-website

git stash            # guarda a meta do Facebook (alteração local do index.html)
git pull             # traz o commit novo (fast-forward)
git stash pop        # reaplica a meta do Facebook

# conferência rápida (ajuste o grep ao seu fix)
grep -c facebook-domain-verification index.html   # esperado: 1
```

> Se o `git stash pop` acusar conflito, resolva no `index.html` (a área do `<head>` vs. o resto do arquivo raramente colidem) e siga.

### 3.3. Rebuild + recriação do container

Como o site ainda **não está no compose** (ver seção 5 para o fix definitivo), o deploy hoje é manual:

```bash
cd /docker/ai4all-website

# 1) build da nova imagem (mesmo nome)
docker build -t docker-ai4all-website .

# 2) guardar o container atual como fallback
docker rename ai4all-website ai4all-website-old
docker stop ai4all-website-old

# 3) recriar a REGRA do Traefik sem risco de o texto virar link no copy/paste
#    (constrói "www" e a crase por código, sem escrever esses caracteres)
RULE=$(python3 -c "d='ai4allplatform.com'; b=chr(96); hosts=[d,'w'*3+'.'+d,d+'.br','w'*3+'.'+d+'.br']; print(' || '.join('Host('+b+h+b+')' for h in hosts))")

# 4) subir o novo container com as MESMAS labels/rede/restart
docker run -d \
  --name ai4all-website \
  --network bridge \
  --restart unless-stopped \
  --label traefik.enable=true \
  --label 'traefik.http.routers.website.entrypoints=websecure' \
  --label "traefik.http.routers.website.rule=$RULE" \
  --label 'traefik.http.routers.website.tls.certresolver=letsencrypt' \
  --label 'traefik.http.services.website.loadbalancer.server.port=80' \
  docker-ai4all-website
```

### 3.4. Verificação

```bash
# conteúdo servido dentro do container
docker exec ai4all-website grep -o 'CNPJ[^<]*' /usr/share/nginx/html/index.html

# a label do Traefik está correta? (testes que NÃO imprimem "www", à prova do artefato de exibição)
docker inspect ai4all-website --format '{{index .Config.Labels "traefik.http.routers.website.rule"}}' | grep -c '\['   # esperado: 0
docker inspect ai4all-website --format '{{index .Config.Labels "traefik.http.routers.website.rule"}}' | tr -d '\n' | md5sum
#   md5 esperado dos 4 hosts corretos: 35d3e9d680cd539b8ccb04c6c1c84c33

docker ps --filter name=ai4all-website   # deve estar "Up"
```

Se estiver tudo certo, remova o fallback:

```bash
docker rm ai4all-website-old
```

Por fim, abra no navegador as **4 URLs** (as duas `www` inclusive) e confirme o fix.

---

## 4. Armadilhas que nos custaram tempo hoje (para não repetir)

1. **O "www virando link" é ilusão de exibição.** O terminal/cliente transforma `www.dominio.com` em `[www.dominio.com](https://www.dominio.com)` **apenas na exibição e no copy/paste**. O dado real (a label) está limpo. **Nunca** confie no texto colado para julgar se a regra está certa — use os testes objetivos: `grep -c '\['` (tem que dar `0`) e o `md5sum` (tem que bater `35d3e9d680cd539b8ccb04c6c1c84c33`). O comando com `python3 ... 'w'*3 ... chr(96)` existe justamente para o copy/paste nunca corromper a regra.

2. **O container do site é "órfão" de compose.** O `docker compose ls` mostra o projeto `docker` (config `/docker/ai4all-platform/infra/docker/docker-compose.yml`), mas esse compose **não** declara mais o serviço `ai4all-website` — só `ai4all-bot`. Por isso `docker compose up -d --build ai4all-website` dá **`no such service`**. Enquanto o item da seção 5 não for feito, o deploy é manual (seção 3.3). O fix definitivo está na seção 5.

3. **O repositório certo é `ai4all-website`, não `AI4ALL-Platform`.** São repos separados. O deploy do site puxa de `/docker/ai4all-website`.

4. **Sem bind mount:** `docker inspect ... {{json .Mounts}}` retorna `[]`. Os arquivos são **assados na imagem** no build — por isso todo deploy exige `docker build` (não adianta só `git pull`).

5. **Segurança:** o remote do repo `AI4ALL-Platform` na VPS tinha um token do GitHub embutido na URL (`https://x-access-token:ghp_...@github.com/...`). Evitar expor esse output; rotacionar o token se ele vazar.

---

## 5. Fix definitivo — colocar o site no docker-compose

Para o próximo deploy ser apenas `docker compose up -d --build ai4all-website` (sem recriar container na mão), adicionar o serviço abaixo ao `docker-compose.yml` do projeto `docker` (`/docker/ai4all-platform/infra/docker/docker-compose.yml`):

```yaml
  ai4all-website:
    build:
      context: /docker/ai4all-website
      dockerfile: Dockerfile
    image: docker-ai4all-website
    container_name: ai4all-website
    restart: unless-stopped
    networks:
      - bridge
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.website.entrypoints=websecure"
      - "traefik.http.routers.website.rule=Host(`ai4allplatform.com`) || Host(`www.ai4allplatform.com`) || Host(`ai4allplatform.com.br`) || Host(`www.ai4allplatform.com.br`)"
      - "traefik.http.routers.website.tls.certresolver=letsencrypt"
      - "traefik.http.services.website.loadbalancer.server.port=80"
```

E garantir, na seção `networks:` do mesmo arquivo, a rede externa `bridge` (mesclar com a seção existente, não duplicar):

```yaml
networks:
  bridge:
    external: true
    name: bridge
```

Depois de mesclar:

```bash
cd /docker/ai4all-platform/infra/docker
docker compose config            # valida a sintaxe e mostra os serviços
docker rm -f ai4all-website      # remove o container manual atual
docker compose up -d --build ai4all-website
```

A partir daí, o deploy do site vira só:

```bash
cd /docker/ai4all-website && git stash && git pull && git stash pop
cd /docker/ai4all-platform/infra/docker && docker compose up -d --build ai4all-website
```

> **Atenção ao integrar:** o compose já contém o serviço `ai4all-bot` e possivelmente uma seção `networks:`. Fazer o merge com cuidado (indentação de 2 espaços, sem duplicar chaves). Rodar `docker compose config` antes do `up` para validar.
