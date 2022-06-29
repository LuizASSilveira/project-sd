# Gerenciador DAS MEI

## Descrição
Serviço de cadastro para gestão do DAS para o microempreendedor, utilizando microsserviços em Docker e nginx para balanceamento de carga.

## Arquitetura
![image](https://user-images.githubusercontent.com/23039988/176545296-b7af5e90-496e-4569-84d6-86863ea31b23.png)

1. Cliente conecta ao servidor (IP 172.18.0.100:80)
2. Nginx redirecionará a requisição para o servidor web com o menor número de conexões ativas (`172.18.0.210:8080`, `172.18.0.220:8080`)
3. Servidor web solicita o serviço ao microsserviço responsável 
4. Os microsserviços pode realizar uma consulta ao banco de dados (`172.18.0.240:3306`), realizar uma consulta externa ao serviço do governo (http://www8.receita.fazenda.gov.br) ou realizar uma chamada de processo remoto para outro microsserviço (`gRPC`).

## Interface

1. login(String email, String password)
2. getBoleto(String cnpj, String data)
3. listStatusCNPJ(String cnpj)
4. tokenIsValid(String token)
5. registerUser(String name, String email, String phone, String cnpj )

## Implementação

### Ciando Subnets
Isolando os containers em uma subnet e atribuindo IPs fixos para cada container. A subnet será `172.18.0.0/16`.

```sh
docker network create --subnet=172.18.0.0/16 dasmei-network
```
#### Balanceador de Carga
Criando uma imagem personalizada do nginx que irá distribuir as requisições recebidas entre os servidores web. O acesso será atravez do IP `172.18.0.100` na porta `80`.

Arquivo de configuração nginx (`nginx-load-balancer.conf`), no qual esta configurado o algoritimo de balanceamento `least_conn` no qual redireciona a requisição para o servidor com o menor número de conexões.

```nginx-load-balancer.conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}

http {
    upstream dasmei.com.br {
        least_conn;
        server 172.18.0.210:8080;
        server 172.18.0.220:8080;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://dasmei.com.br;
        }
    }
}
```

Criando dock file
```Dockerfile.loadbalancer
FROM nginx:stable-alpine
COPY nginx-load-balancer.conf /etc/nginx/nginx.conf
```

Gerando a imagem a partir do Dockerfile acima
```sh
docker build -t load-balancer:1.0 --file Dockerfile.loadbalancer .
```

Executando o nginx
```sh
docker run -d -p 80:80 --rm --net dasmei-network --ip 172.18.0.100 --name load-balancer load-balancer:10
```

### Servidores web
Para servidor web utilizarei o neginx novamente.

```nginx-web-server.conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}

http {
    server {
        listen 8080;
        root /dasmei;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```
```Dockerfile.webserver
FROM nginx:stable-alpine
COPY nginx-web-server.conf /etc/nginx/nginx.conf
```
```sh
docker build -t web-server:1.0 --file Dockerfile.webserver .
```
executando os web server em dois IP's distintos, sendo web-1 no IP `172.18.0.210` e web-2 `172.18.0.220`
```sh
docker run -d --rm --net dasmei-network --ip 172.18.0.210 -v front-end:/dasmei/ --name web-1 web-server:1.0
docker run -d --rm --net dasmei-network --ip 172.18.0.220 -v front-end:/dasmei/ --name web-2 web-server:1.0
```

### Banco de Dados
Será usado o MariaDB como banco de dados:
```sh
docker run -d --rm --net dasmei-network --ip 172.18.0.240 --name mariadb -e MARIADB_USER=user -e MARIADB_PASSWORD=123456 -e MARIADB_ROOT_PASSWORD=@123456 mariadb:10.7.4
```
