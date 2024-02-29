## Konga / Kong / Keycloak: protegendo API com OIDC


### Requisitos

- [**docker**](https://docs.docker.com/install/)
- [**docker-compose**](https://docs.docker.com/compose/overview/)
- [**jq**](https://stedolan.github.io/jq/)
- [**curl** cheatsheet ;)](https://devhints.io/curl)

### Versões instaladas

- Kong 2.7.1 - alpine
- Konga 0.14.7
- Keycloak 17.0.0

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

OpenID é um nível simples de identidade implementado acima do protocolo OAuth 2.0: permite aos seus Clientes verificar a identidade do utilizador final, com base na autenticação realizada por um Authorization Server, bem como obter informações básicas sobre o perfil do utilizador.

Com um Security Token Service (STS), o RP é redirecionado para um STS, que autentica o RP e emite um token de segurança que concede acesso, em vez do aplicativo que autentica diretamente o RP. As declarações são extraídas de tokens e usadas para atividades relacionadas à identidade.

O padrão OpenID define uma situação em que um site cooperante pode atuar como um RP, permitindo ao usuário acessar vários sites usando um conjunto de credenciais. O utilizador beneficia de não ter de partilhar credenciais de acesso com vários sites e os operadores do site colaborador não devem desenvolver o seu próprio mecanismo de acesso.

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

Konga é um painel de administração do Kong. Oferece-nos um painel visual através do qual podemos realizar as configurações do Kong (bem como inspecionar as configurações feitas a partir da linha de comando).

Konga está escutando na porta 1337. Portanto, iniciamos um navegador e apontamos para a url http://192.168.0.223:1337

Para fazermos login no konga, precisaremos registrar uma conta de administrador. Para testes, use credenciais simples e fáceis de lembrar. Para sistemas de produção, utilize senhas que atendam aos padrões de segurança!

Após cadastrar o usuário administrador, será possível efetuar login.

Uma vez logado, precisaremos ativar a conexão com Kong. Digite em "Nome" o valor "kong" e como "Kong Admin URL" o seguinte endereço: http://kong:8001 e depois salve.


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

![Register Service](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/response-criacao-rota-kong.png)


3.2 Anote o ID do serviço registrado (no exemplo é 580b9448-d6b9-47aa-8f42-e275f44b8296) e use-o para fazer a próxima chamada à API do kong que permite adicionar uma rota ao serviço registrado.

![Add Service Route](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/response-de-adi%C3%A7%C3%A3o-de-rota-ao-serv%C3%A7o..png)


### 4. Configuração do Keycloak

O Keycloak estará disponível na url http://192.168.0.223:8180. Você pode fazer login usando credenciais dentro do arquivo docker-compose.yml, (as credenciais padrão são admin/admin).

![Login Keycloak](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-login.png)

Após o login, clique no botão “Adicionar Realm”: este botão aparece quando o mouse passa sobre o nome do realm (Master) no canto superior esquerdo:

![Add Realm Keycloak](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/keycloak-add-realm.png)

Você precisa atribuir um nome ao realm. Para este exemplo, optei pelo nome "central", mas sinta-se à vontade para escolher o nome que melhor se adeque às suas necessidades:

![Add New Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/add-new-realm.png)

Após salvar, você será redirecionado para a página de configurações do realm:

![Created Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/realm-created.png)

Esta página contém muitas abas, cada uma com diversos campos de configuração.

Após a criação do domínio, é necessário adicionar dois clientes:

- Um cliente que será utilizado pelo Kong, por meio do plugin OIDC.

- Outro cliente que será usado para acessar a API através do Kong.

Para criar o primeiro cliente, chamado "kong", selecione “Clientes” no menu da barra lateral esquerda e clique no botão “Criar” localizado no lado direito da página.

![Add Client Private Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/add-client-private-realm.png)


No campo "Cliente ID", insira a string "kong" e depois clique no botão "Salvar" ou "Confirmar" para concluir a criação do cliente.


![Add Client Private Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/config-client-private-realm.png)

Atenção para os campos específicos e a aba onde podemos encontrar o segredo necessário:

- Client Protocol: Escolha "openid-connect", já que esta conta é para OIDC.

- Access Type: Selecione "confidencial". Isso indica que o cliente requer um segredo para iniciar o processo de login. Essa chave será utilizada posteriormente na configuração do Kong OIDC.

- Root Url: Forneça o URL raiz, se aplicável.

- Valid Redirect URLs: Preencha este campo com os URLs válidos para redirecionamento.

Na aba "Credentials", você encontrará o segredo necessário que será utilizado para configurar o Kong OIDC. Preste atenção a esses detalhes ao configurar o cliente.

![Secret Client Private Realm](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/generate-secret-client-private.png)

Agora, vamos criar um segundo cliente com o nome 'application'

![Create Client Public](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/create-client-public-application.png)

Após criar o cliente 'application', proceda com suas configurações

![Config Client Public](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/config-client-public-application.png)

O importante aqui é o tipo de acesso: "público" implica que o processo de login exige as credenciais do usuário para ser concluído.

Agora, vamos criar um usuário que será utilizado posteriormente para autenticação. Para isso, clique no menu lateral esquerdo em "Manage" > "Users", e em seguida, no lado direito da página, clique no botão "Add User"

![Create User Application](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/create-user-application.png)

Atenção ao campo "Email Verified" (você deve ativá-lo, caso contrário o keycloak tentará validar o e-mail do usuário). O usuário ainda não possui uma senha. Então vá na aba "Credentials" e preencha os campos "New password" e "Password Confirmation" com a senha do usuário. Coloque a chave "Temporary" em "Off", caso contrário o keycloak solicitará ao usuário que altere a senha no primeiro login. Clique em "Set Password" para aplicar a credencial.

![Set Password User Application](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/set-password-user-application.png)

### 5. Configuração Kong como cliente Keycloak

Para ativar a funcionalidade do OIDC com o Kong como cliente do Keycloak e permitir a introspecção, é necessário chamar a API Admin Rest do Kong.

A API relevante é a /plugins, que permite adicionar um plugin globalmente ao Kong.

Para configurar o plugin OIDC, você precisará fazer a seguinte requisição:

![Request Kong Plugin OIDC](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/Configura%C3%A7%C3%A3o%20Kong%20como%20cliente%20Keycloak.png)

Se desejar mais detalhes sobre as várias configurações utilizadas neste pedido, visite a página do GitHub do plugin Kong OIDC. Consulte a seção "Uso" para mais informações.

Observe a resposta da solicitação ao definir "bearer_only=yes": com essa configuração, o Kong realizará a introspecção dos tokens sem redirecionar. Isso é útil ao criar um aplicativo ou página da web e desejar ter controle total sobre o processo de login. Na prática, o Kong não redirecionará o usuário para a página de login do Keycloak em caso de solicitação não autorizada, mas responderá com um status 401.


Após a conclusão bem-sucedida da configuração, você pode visualizar a configuração de forma gráfica acessando Konga > Plugins:

![Plugin OIDC](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/plugin-oidc-konga.png)

### Executando os testes

Se realizar uma solicitação diretamente à API usando o Postman ou Curl, notará que agora está protegida e retornará o código de status 401, indicando acesso não autorizado.

![API Protected Kong Keykloak](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/401-kong-keycloak.png)

Agora, vamos gerar o access token para acessar a API protegida, fornecendo as credenciais de usuário e senha da aplicação configuradas no Keycloak.

![API Token Generated](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/geração-token-jwt-keycloak.png)

Após a geração do access token, podemos acessá-la sem problemas.

![API Access](https://raw.githubusercontent.com/RodrigoAntonioCruz/assets/main/Kong/chamada-api-kong-keycloak-sucesso.png)