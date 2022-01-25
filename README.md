> To install the symfony app it is recommended to follow the instructions in this tutorial https://github.com/whernandez/symfony5-dockerization 
after installing the app you can continue here.

## JWT USER AUTHENTICATION
Requirements: knowledge of docker, mysql, and symfony

We will apply the authentication via JWT in two processes, these are:
1. Create the entity and the first user of the App
2. Second, use the LexikJWt Bundle to install the necessary tools to log in using a token.

### Install dependencies
Lets into the container
```bash
make ssh-be 
```
Install required dependencies
```bash
composer require symfony/uid symfony/security-bundle symfony/orm-pack logger
composer require --dev symfony/maker-bundle profiler debug
```

### Create user security settings
```bash
bin/console make:user
```

```bash
The name of the security user class (e.g. User) [User]:
 > 

 Do you want to store user data in the database (via Doctrine)? (yes/no) [yes]:
 > 

 Enter a property name that will be the unique "display" name for the user (e.g. email, username, uuid) [email]:
 > email

 Will this app need to hash/check user passwords? Choose No if passwords are not needed or will be checked/hashed by some other system (e.g. a single sign-on server).

 Does this app need to hash/check user passwords? (yes/no) [yes]:
 > 
```


### Run the migrations to create the database and entity
1. Copy the configuration of the environment variable .env > .env.dev.local
```bash
cp .env .env.dev.local
```

Replace the value of the DATABASE_URL variable
```
DATABASE_URL="database_host://root:your_custom_password@mysql:3306/symfony?serverVersion=mariadb-10.2.25"
```

Set up the database
```
bin/console doctrine:database:create
bin/console make:migration
bin/console doctrine:migrations:migrate
```

### Create user
1. Create a user manually in the database, the password will be 1234, and we are going to generate a hash with the following command:
```
email: test@test.com
roles: ["ROLE_ADMIN","ROLE_USER"]
```
```bash
bin/console security:hash-password 1234
```

### Setup JWT

```bash
composer require "lexik/jwt-authentication-bundle"
```

Generate ssl keys
```bash
bin/console lexik:jwt:generate-keypair
```

Configure your config/packages/security.yaml :

Make sure the firewall login is place before api, and if main exists, put it after api, otherwise you will encounter /api/login_check route not found.

```bash
# Symfony 5.3 and higher
security:
    enable_authenticator_manager: true
    # ...
    
    firewalls:
        login:
            pattern: ^/api/login
            stateless: true
            json_login:
                check_path: /api/login_check
                success_handler: lexik_jwt_authentication.handler.authentication_success
                failure_handler: lexik_jwt_authentication.handler.authentication_failure

        api:
            pattern:   ^/api
            stateless: true
            jwt: ~

    access_control:
        - { path: ^/api/login, roles: PUBLIC_ACCESS }
        - { path: ^/api,       roles: IS_AUTHENTICATED_FULLY }
```

Configure your routing into config/routes.yaml :
```bash
api_login_check:
    path: /api/login_check
```

Exit the container and test the app
```bash
exit
```

Obtain a token with the wrong user, if everything is ok it should respond:
```bash
curl -X POST -H "Content-Type: application/json" http://api.symfony.local/api/login_check -d '{"username":"johndoe","password":"test"}'

{"code":401,"message":"Invalid credentials."}
```

Make the same request but this time using the user created in the database, if everything goes well, it should return the user token:
```bash
curl -X POST -H "Content-Type: application/json" http://api.symfony.local/api/login_check -d '{"username":"test@test.com","password":"1234"}'

{"token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJpYXQiOjE2NDI1NjA1NTksImV4cCI6MTY0MjU2NDE1OSwicm9sZXMiOlsiUk9MRV9BRE1JTiIsIlJPTEVfVVNFUiJdLCJ1c2VybmFtZSI6InRlc3RAdGVzdC5jb20ifQ.RLwC0_0wNHdMO2RBVHvMUjTQIJiLRR2Kgk0TUbjvUQrITAdMMamLxmyvtWaHQ3hKb4F1M7rs4q2Xc5q1kQMQq9j2HNJ4CeQRA1F99dNIaA4piW_ueyiq77wClwNK658K7VJ7WPrJotNYF4CRsYKqWgUbDw2gYmqr-O5BLD1UwTgRYaE59PQ6czkkmO5wHfch0dxgcR3VXckkH7q81ueOI7Cfl7PxMF__zRBPF9TvL-4YjgyoJ9UQSjfXsyoVpObBD0g9q7tCsYKvwdM7MZTox-02XNyst813_GubnIY0kPMCoNbwZ0XhI5GZs6hA1FNhyZuVbmyxuoDTAk1IUUsnDw"}
```

That's it all, mission accomplished!
