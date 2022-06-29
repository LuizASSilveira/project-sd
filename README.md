# Gerenciador DAS MEI

### Descrição
Serviço de cadastro para gestão do DAS para o microempreendedor, utilizando microsserviços em Docker e nginx para balanceamento de carga.

### Arquitetura
![image](https://user-images.githubusercontent.com/23039988/176545296-b7af5e90-496e-4569-84d6-86863ea31b23.png)

1. Cliente conecta ao servidor (IP 172.18.0.100:80)
2. Nginx redirecionará a requisição para o servidor web com o menor número de conexões ativas (172.18.0.210:8080, 172.18.0.220:8080)
3. Servidor web solicita o serviço ao microsserviço responsável 
4. Os microsserviços pode realizar uma consulta ao banco de dados (172.18.0.240:3306), realizar uma consulta externa ao serviço do governo (http://www8.receita.fazenda.gov.br) ou realizar uma chamada de processo remoto para outro microsserviço (gRPC).

### Interface

1. login(String email, String password)
2. getBoleto(String cnpj, String data)
3. listStatusCNPJ(String cnpj)
4. tokenIsValid(String token)
5. registerUser(String name, String email, String phone, String cnpj )
