# Deploy do DesertPB em servidor próprio

Este tutorial assume **Ubuntu com Nginx**, com o projeto em `/var/www/insa-front`
e o site publicado em `/insa-front/`, sem alterar código. Os dados ficam em
`insa-front/public/tiles`.

> Estes passos foram escritos a partir da configuração do projeto, mas ainda não
> foram executados em um servidor.

## O que é necessário

- **Acesso:** SSH com `sudo` e, para usar HTTPS, um domínio apontando para o servidor.
- **Node:** versão 22.12 ou superior, exigida pelo Vite 8.
- **Disco:** cerca de 4 GB livres e 900 mil inodes. Os tiles têm cerca de 420 mil
  arquivos e ocupam 1,7 GB, e o build os duplica temporariamente.

## Três pontos que afetam o deploy

1. **Os tiles não vêm no `git clone`:** `public/tiles/` está no `.gitignore`, então
   é preciso enviá-los do computador local.
2. **O build usa o servidor de tiles atual:** `.env.production` aponta para
   `sistema.sigrural.com.br`. Use um `.env.production.local`, que tem prioridade e
   não é versionado, para não afetar o GitHub Pages.
3. **O build copia `public/` para `dist/`:** os tiles ficariam duplicados. Por isso,
   o passo 3 apaga `dist/tiles` e o Nginx lê direto de `public/tiles`.

## 1. Preparar o servidor

```bash
ssh usuario@SEU_SERVIDOR

df -h /var && df -i /var

sudo apt update
sudo apt install -y nginx git
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

sudo mkdir -p /var/www/insa-front
sudo chown $USER:$USER /var/www/insa-front
git clone git@github.com:marcellobenigno/insa-front.git /var/www/insa-front
```

Se o repositório for privado, o servidor precisa de uma chave SSH autorizada no GitHub.

## 2. Enviar os tiles

No computador local:

```bash
cd ~/code/insa-front/public/tiles
COPYFILE_DISABLE=1 tar -zcf ~/insa_layers.tar.gz insa_layers
scp ~/insa_layers.tar.gz usuario@SEU_SERVIDOR:~/
rm ~/insa_layers.tar.gz
```

Depois, no servidor:

```bash
mkdir -p /var/www/insa-front/public/tiles
cd /var/www/insa-front/public/tiles
rm -rf insa_layers    # só tem efeito em atualizações
tar --warning=no-unknown-keyword -zxf ~/insa_layers.tar.gz
rm ~/insa_layers.tar.gz
```

O `focos_queimadas.geojson` já vai junto, pois fica dentro de `insa_layers/`.

## 3. Gerar o build

No servidor:

```bash
cd /var/www/insa-front
echo 'VITE_TILES_URL=/insa-front/tiles/insa_layers/{z}/{x}/{y}.pbf' > .env.production.local

npm ci
npm run build      # pode demorar alguns minutos, pois copia os tiles
rm -rf dist/tiles  # remove a cópia duplicada
```

## 4. Configurar o Nginx

Crie o arquivo de configuração:

```bash
sudo nano /etc/nginx/sites-available/insa-front
```

Cole o conteúdo abaixo, trocando `SEU_DOMINIO_OU_IP`:

```nginx
server {
    listen 80;
    server_name SEU_DOMINIO_OU_IP;

    location = / { return 302 /insa-front/; }
    location = /insa-front { return 301 /insa-front/; }

    location /insa-front/ {
        alias /var/www/insa-front/dist/;
        index index.html;
    }

    location /insa-front/assets/ {
        alias /var/www/insa-front/dist/assets/;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    location ^~ /insa-front/tiles/ {
        alias /var/www/insa-front/public/tiles/;
        types {
            application/x-protobuf pbf;
            application/geo+json   geojson;
        }
        gzip on;
        gzip_types application/x-protobuf application/geo+json;
        expires 1d;
        log_not_found off;
    }
}
```

Ative o site:

```bash
sudo ln -s /etc/nginx/sites-available/insa-front /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

> ⚠️ **Se o servidor já hospeda outros sites:** não remova o `default`. Nesse caso,
> copie só os blocos `location` para o `server` existente, deixando de fora o
> `location = /` se a raiz já for usada por outro site. Isso vale, por exemplo, para
> o `sistema.sigrural.com.br`.

## 5. Ativar o HTTPS (requer domínio)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d SEU_DOMINIO
```

## 6. Testar

```bash
curl -I http://SEU_DOMINIO_OU_IP/insa-front/
curl -I http://SEU_DOMINIO_OU_IP/insa-front/tiles/insa_layers/8/95/135.pbf
curl -I http://SEU_DOMINIO_OU_IP/insa-front/tiles/insa_layers/focos_queimadas.geojson
```

Os três devem retornar `200`. Depois, abra `/insa-front/#/mapa` no navegador, ligue
uma camada e veja na aba Network se os `.pbf` vêm do seu servidor. Um 404 em tile
fora da área coberta pelos dados é normal.

## Como atualizar

**Só o código:**

```bash
cd /var/www/insa-front
git pull
npm ci
npm run build
rm -rf dist/tiles
```

**Os dados:** gere os tiles de novo localmente com o pipeline, repita o passo 2 e
depois rode os comandos acima, porque `styles.json`, `stats.json` e
`search_index.json` são embutidos no build. Para isso, faça antes o commit e o push
desses arquivos.

## Cuidados

- **`npm run deploy:tiles`:** não use esse comando para este servidor, pois o script
  envia os tiles para `ubuntu@2.25.137.181`, em `/var/www/html/tiles`.
- **`git clean -fdx`:** não rode no servidor, pois ele apaga os tiles e o
  `.env.production.local`.
- **Internet:** o navegador do visitante ainda precisa de acesso externo para carregar
  os mapas base (OSM, Esri, Google), o Bootstrap Icons e o Google Fonts.

## Para publicar na raiz do domínio

Para usar `https://dominio/` sem `/insa-front/`, são três mudanças:

- No `.env.production.local`, use `VITE_TILES_URL=/tiles/insa_layers/{z}/{x}/{y}.pbf`.
- Faça o build com `npm run build -- --base=/`.
- No Nginx, remova as duas regras de redirecionamento e tire `/insa-front` dos
  caminhos dos `location`.
