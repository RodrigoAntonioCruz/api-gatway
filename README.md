## Konga / Kong / Keycloak: protegendo API com OIDC


### Requisitos

- [**docker**](https://docs.docker.com/install/)
- [**docker-compose**](https://docs.docker.com/compose/overview/)
- [**jq**](https://stedolan.github.io/jq/)
- [**curl** cheatsheet](https://devhints.io/curl)

### Versões instaladas

- Kong 2.8.1 - alpine
- Konga 0.14.7
- Keycloak 23.0.3

### Objetivo
O principal objetivo é garantir a segurança por meio da configuração do Kong e do Keycloak para proteger os recursos da API. Para uma compreensão mais detalhada, examinaremos o seguinte fluxo de solicitação:

![Diagrama de fluxo](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/diagrama-kong-keycloak.png)

1 - O aplicativo do usuário envia uma solicitação ao gateway API (kong). No entanto, a solicitação não está autenticada (ou contém uma autenticação inválida).

2 - API do gateway responde ao cliente indicando a falta de autenticação.

3 - Portanto, o aplicativo precisa fazer login. Portanto, ele envia uma solicitação específica de login para o Single Sign On (Keycloak), incluindo as credenciais do usuário e o client-id específico atribuído ao próprio aplicativo.

4 - Se as credenciais forem válidas, o SSO (Keycloak) emite para a aplicação um token (e o token de atualização relacionado), com o qual autenticar as solicitações à API Gateway (Kong)

5 - A aplicação então repete a solicitação adicionando o token válido como autorização

6 - A API do gateway procederá à verificação (por meio de introspecção) se o token em questão corresponde a uma sessão no Single Sign On (Keycloak).

7 - O resultado da introspecção é devolvido a Kong, que tratará da solicitação de inscrição adequadamente

8 - Por fim, o serviço protegido responde com sucesso os recursos solicitados.


Observação:
O aplicativo pode fazer login no keycloak antes mesmo de enviar a primeira solicitação. Na verdade, normalmente é assim, se pensarmos no caso de um aplicativo móvel: uma vez inseridas as credenciais, o usuário pode ter optado por permanecer conectado (então, no máximo, o aplicativo solicitará um novo token válido usando o token de atualização).

---

### Breve introdução ao OIDC
OpenID é uma camada simplificada de identidade que opera sobre o protocolo OAuth 2.0. Ele possibilita que os clientes verifiquem a identidade do usuário final com base na autenticação realizada por um Authorization Server, além de permitir a obtenção de informações básicas sobre o perfil do usuário.

A utilização de um Security Token Service (STS) adiciona uma camada extra de segurança ao processo. Nesse caso, o Resource Provider (RP) é redirecionado para um STS, que autentica o RP e emite um token de segurança, concedendo acesso em vez de o aplicativo autenticar diretamente o RP. As declarações contidas nos tokens são extraídas e utilizadas para diversas atividades relacionadas à identidade.

O padrão OpenID estabelece uma situação na qual um site cooperante pode atuar como um RP, permitindo que o usuário acesse vários sites usando um conjunto único de credenciais. Isso beneficia o usuário, pois ele não precisa compartilhar suas credenciais de acesso com múltiplos sites, enquanto os operadores do site colaborador não precisam desenvolver seu próprio sistema de autenticação.

:point_right: Links úteis

- [Relying Party](https://en.wikipedia.org/wiki/Relying_party)
- [Claims based identity](https://en.wikipedia.org/wiki/Claims-based_identity)
- [OpenID](https://en.wikipedia.org/wiki/OpenID)

### 1- Construção das imagens docker:

1.1 - Este comando constrói a imagem do serviço Kong definido no arquivo `docker-compose.yml`:
```bash
docker-compose build kong
```

1.2 - Este comando inicia o serviço de banco de dados do Kong em segundo plano:
```bash
docker-compose up -d kong-db
```

1.3 - Este comando executa as migrações necessárias no banco de dados do Kong para inicializá-lo:
```bash
docker-compose run --rm kong kong migrations bootstrap
```

1.4 - Este comando executa todas as migrações pendentes no banco de dados do Kong. É usado para aplicar migrações adicionais após a inicialização do banco de dados:
```bash
docker-compose run --rm kong kong migrations up
```

1.5 - Este comando inicia o serviço Kong em segundo plano:
```bash
docker-compose up -d kong
```

1.6 - Este comando lista todos os serviços definidos no arquivo `docker-compose.yml` e seu status atual:
```bash
docker-compose ps
```

1.7 - Este comando faz uma solicitação HTTP para o serviço Kong na porta 8001 (que é a porta padrão da API do Kong) e usa `jq` para formatar a saída JSON e filtrar os plugins disponíveis do Kong que suportam o protocolo OpenID Connect (OIDC):
```bash
curl -s http://localhost:8001 | jq .plugins.available_on_server.oidc
```

1.8 - Este comando inicia o serviço Konga em segundo plano:
```bash
docker-compose up -d konga
```

1.9 - Este comando inicia o serviço de banco de dados do Keycloak em segundo plano:
```bash
docker-compose up -d keycloak-db
```

1.10 - Este comando inicia o serviço do Keycloak em segundo plano:
```bash
docker-compose up -d keycloak
```

1.11 - Por fim, listamos todos os caontainers para saber o seu status atual após a inicialização:
```bash
docker-compose ps
```
#### Importante: Para as configurações seguintes iremos utilizar o endereço de IP. Para isso, será necessário descobri-lo através do terminal. Se você estiver utilizando essa configuração em um servidor externo, utilize o nome do seu host. 

#### Windows:
- **ipconfig**: No Prompt de Comando do Windows, você pode digitar `ipconfig` e pressionar Enter para ver todas as informações de rede, incluindo o endereço IP.

#### macOS e Linux:
- **ifconfig**: Este comando mostra informações detalhadas sobre as interfaces de rede, incluindo o endereço IP. No entanto, em sistemas mais recentes, pode ser necessário usar `sudo ifconfig` para visualizar as informações completas.

- **ip addr**: Este é um comando moderno que também exibe informações sobre as interfaces de rede e o endereço IP associado. É preferido sobre `ifconfig` em distribuições Linux mais recentes.

- **hostname -I**: Este comando exibe apenas o endereço IP do host atual, o que pode ser útil para sistemas com apenas uma interface de rede ativa.


### 2. Configuração do Konga

O Konga é uma interface de administração para o Kong, fornecendo um painel visual que facilita a configuração do Kong, além de permitir a inspeção das configurações feitas através da linha de comando.

Para acessar o Konga, basta abrir um navegador e apontar para a URL http://192.168.0.133:1337, já que o Konga está escutando na porta 1337.

Para fazer login no Konga, é necessário registrar uma conta de administrador. Para testes, recomenda-se utilizar credenciais simples e fáceis de lembrar. No entanto, em sistemas de produção, é crucial utilizar senhas que atendam aos padrões de segurança.

Após cadastrar o usuário administrador e efetuar o login, será possível ativar a conexão com o Kong. Para isso, basta inserir "kong" como nome e definir o endereço "Kong Admin URL" como http://kong:8001, e então salvar as configurações.

### Criação usuário administrativo no Konga

![Konga Create User](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/criação-admin-konga.png)

### Login com usuário administrativo no Konga

![Konga Login User](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/login-konga.png)

### Configuração inicial do Konga

![Konga Start Config](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/config-inicial-konga-welcome.png)

### Após a configuração inicial estará disponível para uso

![Konga Dashboard](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/dashboard-konga.png)


Neste ponto teremos nossa instância do Konga pronta para uso!

### 3. Criação do serviço e rota no kong

Para realizar testes no sistema, faremos uso da [Fake Store Api](https://fakestoreapi.com/). Este serviço simula uma API, proporcionando um ambiente controlado para testar solicitações HTTP e suas respectivas respostas. É uma ferramenta útil para validar a integração e o comportamento do sistema em diferentes cenários.

3.1 Essa solicitação registra nosso serviço a ser protegido no Kong. Utilize o seguinte comando cURL:

```bash
curl -s -X POST http://192.168.0.133:8001/services \
    -d name=ms-products \
    -d url=https://fakestoreapi.com/products \
    | jq
{
    "connect_timeout": 60000,
    "host": "fakestoreapi.com"
}
```
- Se a requisição for bem-sucedida, você receberá a seguinte resposta:
```bash
{
  "created_at": 1709401849,
  "updated_at": 1709401849,
  "protocol": "https",
  "read_timeout": 60000,
  "write_timeout": 60000,
  "host": "fakestoreapi.com",
  "tls_verify": null,
  "tls_verify_depth": null,
  "enabled": true,
  "retries": 5,
  "id": "de01eaa8-5502-452c-9dcf-b06734f30e1f",
  "name": "ms-products",
  "connect_timeout": 60000,
  "path": "/products",
  "port": 443,
  "tags": null,
  "ca_certificates": null,
  "client_certificate": null
}
```

3.2 Anote o ID do serviço registrado (no exemplo é 580b9448-d6b9-47aa-8f42-e275f44b8296) e use-o para fazer a próxima chamada à API do kong que permite adicionar uma rota ao serviço registrado.

```bash
curl -s -X POST http://192.168.0.133:8001/services/42f8abea-b857-4c2d-8a2c-bb0c15a4597f/routes -d "paths[]=/products" \
    | python -mjson.tool
{
    "destinations": null,
    "hosts": null,
    "methods": null,
    "name": null,
    "paths": [
        "/products"
    ]
}
```
- Se a requisição for bem-sucedida, você receberá a seguinte resposta:
```bash
{
    "created_at": 1709401983,
    "updated_at": 1709401983,
    "paths": [
        "/products"
    ],
    "methods": null,
    "sources": null,
    "request_buffering": true,
    "response_buffering": true,
    "hosts": null,
    "https_redirect_status_code": 426,
    "headers": null,
    "preserve_host": false,
    "regex_priority": 0,
    "service": {
        "id": "de01eaa8-5502-452c-9dcf-b06734f30e1f"
    },
    "id": "08049b21-a5c1-45fc-bcff-5be82cf84a4b",
    "name": null,
    "snis": null,
    "protocols": [
        "http",
        "https"
    ],
    "strip_path": true,
    "tags": null,
    "destinations": null,
    "path_handling": "v0"
}
```


### 4. Configuração do Keycloak

O Keycloak estará disponível na url http://192.168.0.133:8180. Você pode fazer login usando credenciais dentro do arquivo docker-compose.yml, (as credenciais padrão são admin/admin).

![Login Keycloak](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-login.png)

Após o login, clique no botão “Adicionar Realm”: este botão aparece quando o mouse passa sobre o nome do realm (Master) no canto superior esquerdo:

Você precisa atribuir um nome ao realm. Para este exemplo, optei pelo nome "central", mas sinta-se à vontade para escolher o nome que melhor se adeque às suas necessidades:

![Add Realm Keycloak](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-add-new-realm.png)

Após salvar, você será redirecionado para a página de configurações do realm:

![Created Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-realm-created.png)

Esta página contém muitas abas, cada uma com diversos campos de configuração.

Após a criação do domínio, é necessário adicionar dois clientes:

- Um cliente que será utilizado pelo Kong, por meio do plugin OIDC.

- Outro cliente que será usado para acessar a API através do Kong.

Para criar o primeiro cliente, chamado "kong", selecione “Clients” no menu da barra lateral esquerda e clique no botão “Create client” localizado a esquerda do centro da página.

![Create Client Private Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/create-client-private-realm-1.png)

Após nomear o cliente, avance para o próximo passo. Note que precisamos criar um cliente privado para comunicação via OIDC. Portanto, neste passo, habilitaremos a opção 'Client authentication'. Quando esta opção está ativada, o tipo OIDC é configurado como acesso confidencial. Quando está desativada, o tipo de acesso é definido como público.

![Create Client Private Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/create-client-private-realm-2.png)

Neste passo, definimos a 'Root URL' se aplicável, 'Valid redirect URIs' e salvamos nosso cliente.

![Create Client Private Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/create-client-private-realm-3.png)

Por fim, se necessário alterar alguma informação, verifique novamente e então salve as alterações.

![Create Client Private Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/create-client-private-realm-4.png)

Na aba "Credentials", você encontrará o segredo necessário que será utilizado para configurar o Kong OIDC. Preste atenção a esses detalhes ao configurar o cliente.

![Secret Client Private Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-client-private-secret.png)

Agora, vamos criar um segundo cliente com o nome 'application'

![Create Client Public](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-client-public-realm-1.png)

Avance para o próximo passo, agora estamos criando um cliente público, ele servirá para acessar a API através do Kong. Nesse contexto as configurações necessárias podem variar neste caso.

![Create Client Public](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-client-public-realm-2.png)

Para esse cliente em específico, não haverá configurações de URL, deixamos em branco.

![Create Client Public](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-client-public-realm-3.png)

Finalizando, conferimos e se necessário alteramos e salvamos as alterações.
![Create Client Public](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-client-public-realm-4.png)

Agora, vamos criar um usuário que será utilizado posteriormente para autenticação. Para isso, clique no menu lateral esquerdo em "Manage" > "Users", e em seguida, no lado direito da página, clique no botão "Add User"

![Create User Application](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-user-create.png)

Atenção ao campo "Email Verified" (você deve ativá-lo, caso contrário o keycloak tentará validar o e-mail do usuário). O usuário ainda não possui uma senha. Então vá na aba "Credentials" e preencha os campos "New password" e "Password Confirmation" com a senha do usuário. Coloque a chave "Temporary" em "Off", caso contrário o keycloak solicitará ao usuário que altere a senha no primeiro login. Clique em "save" para aplicar a credencial.

![Set Password User Application](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-user-set-password-1.png)

Após clicar em save o sistema pergunta se "Você tem certeza de que deseja definir a senha para o usuário"
![Set Password User Application](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-user-set-password-2.png)

Caso sua resposta seja "Save password" você terá a senha persistida para o usuário
![Set Password User Application](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-user-set-password-3.png)

### 5. Configuração Kong como cliente Keycloak

Para ativar a funcionalidade do OIDC com o Kong como cliente do Keycloak e permitir a introspecção, é necessário chamar a API Admin Rest do Kong.

A API relevante é /plugins, que permite adicionar um plugin globalmente ao Kong.

Para configurar o plugin OIDC, você precisará fazer a seguinte requisição:

```bash
curl -s -X POST http://192.168.0.133:8001/plugins \
  -d name=oidc \
  -d config.client_id=kong \
  -d config.client_secret=GspsUKQJgDZIDKG4N1Mdd120TmtSBVwP \
  -d config.bearer_only=yes \
  -d config.realm=central \
  -d config.introspection_endpoint=http://192.168.0.133:8180/realms/central/protocol/openid-connect/token/introspect \
  -d config.discovery=http://192.168.0.133:8180/auth/realms/central/.well-known/openid-configuration \
  | jq
```
- Se a requisição for bem-sucedida, você receberá a seguinte resposta:

```bash
{
  "created_at": 1709402018,
  "service": null,
  "id": "ee5b8bb5-af72-4e3c-9acd-77d04dd32ec4",
  "name": "oidc",
  "protocols": [
    "grpc",
    "grpcs",
    "http",
    "https"
  ],
  "consumer": null,
  "route": null,
  "tags": null,
  "enabled": true,
  "config": {
    "logout_path": "/logout",
    "bearer_only": "yes",
    "realm": "central",
    "scope": "openid",
    "client_id": "kong",
    "filters": null,
    "ssl_verify": "no",
    "redirect_after_logout_uri": "/",
    "response_type": "code",
    "session_secret": null,
    "introspection_endpoint": "http://192.168.0.133:8180/realms/central/protocol/openid-connect/token/introspect",
    "client_secret": "GspsUKQJgDZIDKG4N1Mdd120TmtSBVwP",
    "discovery": "http://192.168.0.133:8180/auth/realms/central/.well-known/openid-configuration",
    "introspection_endpoint_auth_method": null,
    "redirect_uri_path": null,
    "recovery_page_path": null,
    "token_endpoint_auth_method": "client_secret_post"
  }
}
```

Se desejar mais detalhes sobre as várias configurações utilizadas neste pedido, visite a página do GitHub do plugin Kong OIDC. Consulte a seção "Uso" para mais informações.

Observe a resposta da solicitação ao definir "bearer_only=yes": com essa configuração, o Kong realizará a introspecção dos tokens sem redirecionar. Isso é útil ao criar um aplicativo ou página da web e desejar ter controle total sobre o processo de login. Na prática, o Kong não redirecionará o usuário para a página de login do Keycloak em caso de solicitação não autorizada, mas responderá com um status 401.


Após a conclusão bem-sucedida da configuração, você pode visualizar a configuração de forma gráfica acessando Konga > Plugins:

![Plugin OIDC](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/plugin-oidc-konga.png)

### Executando os testes

Se realizar uma solicitação diretamente à API usando o Postman ou Curl, notará que agora está protegida e retornará o código de status 401, indicando acesso não autorizado.

![API Protected Kong Keykloak](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-access-unauthorized.png)

Agora, vamos gerar o access token para acessar a API protegida, fornecendo as credenciais de usuário e senha da aplicação configuradas no Keycloak.

![API Token Generated](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-access-token-generate.png)

Após a geração do access token, podemos acessá-la sem problemas.

![API Access](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-access-succsess.png)