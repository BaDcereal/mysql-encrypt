# Criptografia de dados em Banco de Dados MySQL usando InnoDB Data-at-Rest Encryption.  

### _Objetivo:_  

#### Este artigo tem como objetivo orientar a implementação de criptografia nas tabelas do tipo InnoDB, não entrarei em detalhes de como instalar o Docker, as etapas descritas a seguir supõem que você já tem o Docker instalado e configurado na sua máquina, este artigo também pode ser utilizado em MySQL Server sem o uso do Docker.

### _Introdução:_  

InnoDB oferece suporte à criptografia de dados em repouso “Data-at-Rest Encryption“ para tablespaces de arquivo por tabela, tablespaces gerais mysql , tablespace do sistema, redo logs e undo logs. Você pode definir um padrão de criptografia para esquemas e espaços de tabela gerais, isso permite que os DBAs controlem se as tabelas criadas nesses esquemas e espaços de tabela são criptografadas.  

### _Sobre criptografia de dados Data-at-Rest Encryption:_  

InnoDB usa uma arquitetura de chave de criptografia de duas camadas, consistindo em uma chave mestra de criptografia e chaves de espaço de tabela. Quando um espaço de tabela é criptografado, uma chave de espaço de tabela é criptografada e armazenada no cabeçalho do espaço de tabela. Quando um aplicativo ou usuário autenticado deseja acessar dados criptografados do espaço de tabela, InnoDB usa uma chave mestra de criptografia para descriptografar a chave do espaço de tabela. A versão descriptografada de uma chave de espaço de tabela nunca muda, mas a chave mestra de criptografia pode ser alterada conforme necessário. Esta ação é conhecida como rotação de chave mestra.  

O recurso de criptografia de dados em repouso depende de um componente de chaveiro ou plug-in para gerenciamento de chave mestra de criptografia.    

Todas as edições do MySQL fornecem um **_component_keyring_file_** componente que armazena dados do chaveiro em um arquivo local no host do servidor.

O MySQL Server suporta um chaveiro que permite que componentes internos do servidor e plug-ins armazenem com segurança informações confidenciais para recuperação posterior. A implementação compreende estes elementos:  
Componentes de chaveiro e plug-ins que gerenciam um armazenamento de apoio ou se comunicam com um back-end de armazenamento. O uso do chaveiro envolve a instalação de um dentre os componentes e plug-ins disponíveis. Os componentes e plug-ins do chaveiro gerenciam dados do chaveiro, mas são configurados de maneira diferente e podem ter diferenças operacionais.

Os componentes do chaveiro disponíveis são: **_component_keyring_file_**, **_component_keyring_encrypted_file_** e **_component_keyring_oci_**. Neste artigo usaremos o **_component_keyring_file_** que permite armazenar dados do chaveiro em um arquivo local no host do servidor e está disponível nas distribuições MySQL Community Edition e MySQL Enterprise Edition. Não entrarei em detalhes sobre os demais componentes porque eles só estão disponíveis na versão MySQL Enterprise Edition.  

> [!WARNING]
> Para o gerenciamento de chaves de criptografia, os componentes **_component_keyring_file_** e **_component_keyring_encrypted_file_** não se destinam a ser uma solução de conformidade regulatória. Padrões de segurança como PCI, FIPS e outros exigem o uso de sistemas de gerenciamento de chaves para proteger e gerenciar chaves de criptografia em cofres de chaves ou módulos de segurança de hardware (HSMs). 

### _Pré-requisitos de criptografia_:  

Um componente ou plug-in de chaveiro deve ser instalado e configurado na inicialização. O carregamento antecipado garante que o componente ou plug-in esteja disponível antes da inicialização do InnoDB.  
Apenas um componente de chaveiro ou plug-in deve ser ativado por vez. A ativação de vários componentes ou plug-ins de chaveiro não é suportada e os resultados podem não ser os esperados.

> **IMPORTANTE:**  
> Depois que os tablespaces criptografados são criados em uma instância do MySQL, o componente de chaveiro ou plug-in que foi carregado durante a criação do tablespace criptografado deve continuar a ser carregado na inicialização. Não fazer isso resultará em erros ao iniciar o servidor e durante a recuperação da tabela InnoDB.  
Ao criptografar dados de produção, certifique-se de tomar medidas para evitar a perda da chave mestra de criptografia. Se a chave mestra de criptografia for perdida, os dados armazenados nos arquivos criptografados do espaço de tabela serão irrecuperáveis.  
Se você usar o componente **_component_keyring_file_** ou, **_component_keyring_encrypted_file_** crie um backup do arquivo de dados do anel de chaves imediatamente após criar o primeiro espaço de tabela criptografado, antes da rotação da chave mestra e após a rotação da chave mestra.

----------

### Instalação do Componente - Parte 1:  
Vamos começar criando um container com o MySQL, se você já tem o MySQL Server instalado avance para a etapa de configuração.

1. Vamos usar a última versão do mysql disponível no Docker hub:

```bash
┌──(user@localhost)-[~]
└─$ mkdir mysql-encrypt

┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ cd mysql-encrypt

┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker pull mysql:latest
```

2. Crie o arquivo **docker-compose.yml** com o conteúdo abaixo:

```bash
services:
  # MySQL
  db:
    image: mysql
    container_name: mysql
    ports: 
        - "3306:3306"
    environment:
        MYSQL_DATABASE: ${DB_DATABASE}
        MYSQL_USER: ${DB_USERNAME}
        MYSQL_PASSWORD: ${DB_PASSWORD}
        MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD} 
        SERVICE_NAME: mysql
        SERVICE_TAGS: mysql-test
    command:
      - --bind-address=0.0.0.0
    volumes:
        - ./dbdata:/var/lib/mysql/
        - ./mysql/my.cnf:/etc/mysql/my.cnf
        # - ./mysql/plugins_conf/mysqld.my:/usr/sbin/mysqld.my:ro
        # - ./mysql/plugins_conf/component_keyring_file.cnf:/usr/lib64/mysql/plugin/component_keyring_file.cnf:rw    
        # - ./mysql/plugins_conf/component_keyring_file:/var/lib/mysql-keyring/component_keyring_file:rw
    restart: unless-stopped
```

3. Crie o arquivo **.env** com o conteúdo abaixo usando as suas definições:

```
DB_DATABASE=testcrypt           # adicione o seu banco de dados
DB_USERNAME=usuario             # adicione o seu usuario
DB_PASSWORD=password            # adicione a sua senha de usuário
DB_ROOT_PASSWORD=passwordroot   # adicione a senha do root
```

4. Crie o diretorio mysql:
   
```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ mkdir mysql
```

5. Dentro do diretório mysql crie o arquivo **my.cnf** que será o arquivo de configuração do mysql:

```
[mysqld]
#0 para UNIX, 1 para todos os sistemas, 2 OSX
lower_case_table_names=1
```

6. Em seguida execute o container:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker compose up
```

7. Abra outro terminal e execute o comando para fazer o login no MySQL como root:  
> Quando for solicitado digite sua senha de root.  

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -it mysql mysql -uroot -p
Enter password:
```

8. Verifique os banco de dados disponíveis:  
> Selecione o banco de dados **testcrypt** ou o banco de dados que você configurou no arquivo **.env** do passo 3:

~~~~sql
mysql> show databases;
mysql> use testcrypt;
~~~~

9. Vamos criar uma tabela users de exemplo:  
> Copie e cole o código abaixo no prompt do MySQL.

~~~~sql
CREATE TABLE `users` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `email` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `email_verified_at` timestamp NULL DEFAULT NULL,
  `password` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `remember_token` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `users_email_unique` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
~~~~

10. Agora vamos inserir alguns registros na tabela:  
> Copie e cole o código abaixo no prompt do MySQL.

~~~~sql
INSERT INTO `testcrypt`.`users` (`name`, `email`, `email_verified_at`, `password`, `remember_token`, `created_at`, `updated_at`) 
VALUES ('João', 'joao@email.com', NULL, '12345678', NULL , now(), now());
INSERT INTO `testcrypt`.`users` (`name`, `email`, `email_verified_at`, `password`, `remember_token`, `created_at`, `updated_at`) 
VALUES ('Maria', 'maria@email.com', NULL, '12345678', NULL , now(), now());
INSERT INTO `testcrypt`.`users` (`name`, `email`, `email_verified_at`, `password`, `remember_token`, `created_at`, `updated_at`) 
VALUES ('José', 'jose@email.com', NULL, '12345678', NULL , now(), now());
~~~~

11. Agora vamos fazer um teste para verificar os dados inseridos na tabela:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ cat dbdata/testcrypt/users.ibd
```

12. Veja um trecho da saída do comando cat:

```bash
                                               supremum
                                                      �Joaojoao@email.com12345678fiؾfiB
                                                                                       �Mariamaria@email.com12345678fi�fi �m
               �oséjose@email.com12345678fi�Qfi�(�}
                                                   @Joãojoao@email.com12345678fiؾfiؾpcm���=Hck���������;AE�̀�
                                                                                                            ��2nfimum
        supremum9joao@email.com��maria@email.com ��jose@email.compcHck�;A%    
```

13. Observe que alguns dados podem ser visualizados facilmente e com o uso de ferramentas de leitura de arquivos .ibd você poderá ver a estrutura da tabela completa com os dados.

14. Vamos agora criar um backup do banco de dados:
> O banco de dados usado no exemplo é o **testcrypt** se você configurou no arquivo **.env** do passo 3 com outro nome de banco de dados altere a linha de código abaixo.
```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -i mysql /usr/bin/mysqldump -uroot -p testcrypt > backup.sql
```

15. O cursor ficará parado esperando o password do root, digite o password e tecle Enter, um arquivo backup.sql será gerado.

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -i mysql /usr/bin/mysqldump -uroot -p testcrypt > backup.sql
Enter password:
```

16. Faça novamente o login no MySQL como root com o comando:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -it mysql mysql -uroot -p
```

17. Em seguida use os comandos para verificar a existência do componente keyring e das chaves:
> Perceba que o retorno será vazio.

~~~~sql
mysql> SELECT * FROM performance_schema.keyring_component_status;
Empty set (0.01 sec)
mysql> SELECT * FROM performance_schema.keyring_keys;
Empty set (0.00 sec)
~~~~

18. Precisamos do diretório que está armazenado na variável **secure_file_priv**, para isso execute:
> O cursor ficará parado esperando o password do root, digite o password e tecle Enter, um arquivo backup.sql será gerado.
```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -i mysql mysql -uroot -p -e "SHOW VARIABLES LIKE 'secure_file_priv';"
Enter password:
```

19. O retorno será similar ao seguinte:

```bash
Variable_name   Value
secure_file_priv        /var/lib/mysql-files/
```

20. Agora precisamos de um script para alterar o status de encriptação da tabela, execute os comandos no terminal, observe o diretório **/var/lib/mysql-files** logo depois de INTO OUTFILE:  
> Se a saída do seu comando retornar um valor diferente coloque esse valor retornado.

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -i mysql mysql -uroot -p -e "SET @dbname = 'testcrypt'; SELECT CONCAT('ALTER TABLE ', table_name, ' ENCRYPTION=\"Y\";') INTO OUTFILE '/var/lib/mysql-files/enable_encryption.sql' FROM information_schema.tables WHERE table_schema = @dbname AND table_type = 'BASE TABLE';"
Enter password:

┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker cp mysql:/var/lib/mysql-files/enable_encryption.sql .
```

21. Pronto você está com o arquivo de script **enable_encryption.sql** que foi gerado para alterar o status da tabela.

22. Já temos o backup do banco de dados e o script para alterar as tabelas, agora vamos excluir o banco de dados para aplicar a encriptição:
> O cursor ficará parado esperando o password do root.
```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -i mysql mysql -u root -p -e "DROP DATABASE testcrypt";
Enter password:
```

----------

### Instalação do componente - Parte 2:  

A primeira etapa na instalação de um componente de chaveiro é escrever um manifesto que indique qual componente carregar. Durante a inicialização, o servidor lê um arquivo de manifesto global ou um arquivo de manifesto global emparelhado com um arquivo de manifesto local.  

O servidor tenta ler seu arquivo de manifesto global no diretório onde o servidor está instalado.  
Se o arquivo de manifesto global indicar o uso de um arquivo de manifesto local, o servidor tentará ler seu arquivo de manifesto local no diretório de dados.  
Embora os arquivos de manifesto globais e locais estejam localizados em diretórios diferentes, o nome do arquivo é **mysqld.my** em ambos os locais.  
Não é um erro não existir um arquivo de manifesto. Nesse caso, o servidor não tenta carregar nenhum componente associado ao arquivo.

1. Crie o diretório plugins_conf dentro do diretório mysql:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ cd mysql

┌──(user@localhost)-[/home/user/mysql-encrypt/mysql]
└─$ mkdir plugins_conf

┌──(user@localhost)-[/home/user/mysql-encrypt/mysql]
└─$ cd ..
```

2. Dentro do diretório plugins_conf crie o arquivo mysqld.my com o seguinte conteúdo:

```
{
  "components": "file://component_keyring_file"
}
```

3. Dadas as propriedades do arquivo de configuração anterior, para configurar **component_keyring_file**, crie um arquivo de configuração global denominado **_component_keyring_file.cnf_** no diretório onde o arquivo de biblioteca **component_keyring_file** está instalado e, opcionalmente, crie um arquivo de configuração local, também denominado **_component_keyring_file.cnf_**, no diretório de dados. As instruções a seguir assumem que um arquivo de dados de chaveiro chamado **/usr/local/mysql/keyring/component_keyring_file** deve ser usado no modo de **leitura/gravação**.
Para usar apenas um arquivo de configuração global, crie o arquivo **_component_keyring_file.cnf_** no diretório plugins_conf, o conteúdo do arquivo é semelhante a este:

```
{
    "path": "/var/lib/mysql-keyring/component_keyring_file", 
    "read_only": false 
}
```

4. Ainda no diretório plugins_conf crie o arquivo **component_keyring_file** sem nenhum conteúdo.

```
No arquivo docker-compose.yml descomente “remova o #” as linhas a seguir:
- ./mysql/plugins_conf/mysqld.my:/usr/sbin/mysqld.my:ro
- ./mysql/plugins_conf/component_keyring_file.cnf:/usr/lib64/mysql/plugin/component_keyring_file.cnf:rw    
- ./mysql/plugins_conf/component_keyring_file:/var/lib/mysql-keyring/component_keyring_file:rw
```

3. Pare o container "Ctrl+c" do mysql, remova o container e de start novamente:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker compose down

┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker compose up
```

4. Após o reinicio do conteiner faça o login no mysql como root e verifique se o componente foi carregado:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -it mysql mysql -uroot -p
Enter password:
```

~~~~sql
mysql> SELECT * FROM performance_schema.keyring_component_status;
~~~~

5. O retorno deve ser similar ao abaixo:

~~~~sql
+---------------------+-----------------------------------------------+
| STATUS_KEY          | STATUS_VALUE                                  |
+---------------------+-----------------------------------------------+
| Component_name      | component_keyring_file                        |
| Author              | Oracle Corporation                            |
| License             | GPL                                           |
| Implementation_name | component_keyring_file                        |
| Version             | 1.0                                           |
| Component_status    | Active                                        |
| Data_file           | /var/lib/mysql-keyring/component_keyring_file |
| Read_only           | No                                            |
+---------------------+-----------------------------------------------+
8 rows in set (0.01 sec)
~~~~

----------

### Funções de gerenciamento de chaves do chaveiro de uso geral.  
O MySQL Server também inclui uma interface SQL para gerenciamento de chaves de chaveiro, implementada como um conjunto de funções de uso geral que acessam os recursos fornecidos pelo serviço de chaveiro interno. As funções do chaveiro estão contidas em um arquivo de biblioteca de plugins, que também contém um **_keyring_udf_** plugin que deve ser habilitado antes da invocação da função.

----------

### Instalação do Chaveiro:  
> Para instalar o **_keyring_udf_** plugin e as funções do chaveiro, use as instruções INSTALL PLUGIN e CREATE FUNCTION, ajustando o .so sufixo para sua plataforma conforme necessário.

1. Faça o login no Mysql como root:
   
```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -it mysql mysql -uroot -p
Enter password:
```

2. Execute os comandos abaixo:
   
~~~~sql
INSTALL PLUGIN keyring_udf SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_generate RETURNS INTEGER
  SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_fetch RETURNS STRING
  SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_length_fetch RETURNS INTEGER
  SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_type_fetch RETURNS STRING
  SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_store RETURNS INTEGER
  SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_remove RETURNS INTEGER
  SONAME 'keyring_udf.so';
~~~~

> Para criar uma nova chave aleatória e armazená-la no chaveiro, chame **_keyring_key_generate()_**, passando para ela um ID da chave, junto com o tipo de chave (método de criptografia) e seu comprimento em bytes.  

3. Use o comando no prompt do MySQL:

~~~~sql
mysql> SELECT keyring_key_generate('MyKey', 'AES', 32);
~~~~

4. O retorno deve ser similar ao abaixo:

~~~~sql
+------------------------------------------+
| keyring_key_generate('MyKey', 'AES', 32) |
+------------------------------------------+
|                                        1 |
+------------------------------------------+
1 row in set (0.00 sec)
~~~~

5. Para verificar se a chave foi criada use o comando:

~~~~sql
mysql> SELECT * FROM performance_schema.keyring_keys;
~~~~

6. O retorno deve ser similar ao abaixo:

~~~~sql
+--------+----------------+----------------+
| KEY_ID | KEY_OWNER      | BACKEND_KEY_ID |
+--------+----------------+----------------+
| MyKey  | root@localhost |                |
+--------+----------------+----------------+
1 row in set (0.00 sec)
~~~~

7. Agora abra o arquivo **component_keyring_file** que esta no diretório **mysql/plugins_conf**, verifique se ele possui conteúdo. Se sim parabéns você instalou o componente de criptografia com sucesso. 

> **IMPORTANTE:**  
> **Faça um backup desse arquivo que é a sua chave, lembre-se a perda desse arquivo impossibilita o acesso as tabelas que estão criptografadas**.

8. Agora vamos voltar os dados salvos para o banco de dados. No terminal execute os comandos abaixo:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -i mysql mysql -uroot -p -e "CREATE DATABASE testcrypt;"
Enter password:
```

9. Depois com o usuario e senha que você configurou no seu arquivo **.env** restaure o banco de dados:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ cat backup.sql | docker exec -i mysql mysql -u usuario --password='password' testcrypt
```

10. Faça o login no Mysql como root:
   
```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -it mysql mysql -uroot -p
Enter password:
```

11. Execute o comando:

~~~~sql
mysql> SELECT TABLE_NAME, CREATE_OPTIONS FROM information_schema.tables WHERE table_schema = 'testcrypt';
~~~~

12. Observe no retorno que a tabela não possui a opção de criptografia habilitada:

~~~~sql
+------------+----------------+
| TABLE_NAME | CREATE_OPTIONS |
+------------+----------------+
| users      |                |
+------------+----------------+
1 row in set (0.00 sec)
~~~~

13. Vamos executar o script **enable_encryption.sql** criado no passo 20 "Instalação do Componente - Parte 1" para habilitar a criptografia das tabelas. Execute o comando no terminal:
> Use o usuario e senha que você configurou no seu arquivo **.env**:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ cat enable_encryption.sql | docker exec -i mysql mysql -u usuario --password='password' testcrypt
```

14. Faça o login no Mysql como root:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -it mysql mysql -uroot -p
Enter password:
```

15. Execute o comando abaixo e observe o retorno, veja que a opção ENCRYPTION='Y' apareceu nas tabelas listadas:

~~~~sql
mysql> SELECT TABLE_NAME, CREATE_OPTIONS FROM information_schema.tables WHERE table_schema = 'testcrypt';

+------------+----------------+
| TABLE_NAME | CREATE_OPTIONS |
+------------+----------------+
| users      | ENCRYPTION='Y' |
+------------+----------------+
1 row in set (0.00 sec)
~~~~

16. Por último você precisa definir a variável **default_table_encryption** para habilitar a criptografia para novas tabelas criadas.

> **IMPORTANTE:**  
> Essa variável só habilita a criptografia em tabelas criadas após a configuração da variável, para alterar uma tabela já existente você precisa fazer isso manualmente, ou usando um script como o **enable_encryption.sql** que geramos anteriormente no passo 20 "Instalação do Componente - Parte 1"  

17. Execute o comando abaixo no prompt do MySQL para definir a variável **default_table_encryption**:

~~~~sql
mysql> SET GLOBAL default_table_encryption = 'ON';
~~~~

18. Depois execute o comando para verificar se a variavel foi alterada para ON:

~~~~sql
mysql> SHOW VARIABLES LIKE 'default_table_encryption';
~~~~

19. Se o retorno do comando for similar ao abaixo você precisa executar o próximo passo, caso contrario avance para o passo seguinte:

~~~~sql
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| default_table_encryption | OFF   |
+--------------------------+-------+
1 row in set (0.00 sec)
~~~~

20. Pare o container "Ctrl+c" e use o comando abaixo para remover o container:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker compose down
```

21. Em seguida abra o arquivo **_my.cnf_** localizado no diretório mysql e adicione a linha abaixo:

```
default_table_encryption = ON
```

22. Salve o arquivo feche e execute o container novamente:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker compose up
```

23. Faça o login no Mysql como root:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ docker exec -it mysql mysql -uroot -p
Enter password:
```

24. Execute o comando abaixo:

~~~~sql
mysql> SHOW VARIABLES LIKE 'default_table_encryption';
~~~~

25. O retorno deve ser similar ao mostrado abaixo:
> Veja que o valor da variável deve ser ON.

~~~~sql
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| default_table_encryption | ON    |
+--------------------------+-------+
1 row in set (0.00 sec)
~~~~

26. Agora vamos testar novamente visualizar o conteúdo da tabela user com o comando:

```bash
┌──(user@localhost)-[/home/user/mysql-encrypt]
└─$ cat dbdata/testcrypt/users.ibd
```

27. Veja que agora é impossível ver qualquer tipo de dado da tabela user.

```bash
       R�K��FJ��!pzW<����w@�ַ|        �Z0c{��AѺ,��G7�Z�h��I�@�$�l�^#?�q!��t�CY{uټ���f��<o'.��>M��`G�7p��b�X��V���� ����0�v����ϴ��
|Y��h�mS�IE*ZάNA���H?�!y.^�́�V��LuhE�b1���\ɓ���^�!�֘y����)&rW5J�g31h��Q��(�a;�覆A�@ї;>�,�!v��D�?�Vv)V�Y�J�����̬
���I5τB�g����(C��0-�R}8��=�#�i��������,���M�|�z0�ퟀX"Ĺ�s��ݩ�A�`�ÀO�lY��G��c.�'����u���,E�i3F+s�\��Q$�
                                                                                                     0U�(&䋊���c@C�R���
          :��fc�բ�*9;R���N#k64  �)�|
�b4�Wg��`?����<��W9�`$y�c�ע�r[#�^�Y.�������-�C�*�.��F�]�
ΦI2'�C���,Oj��) ��4���7��kЯ�L�D~ǽG��n�ܺK�NB}�!��uF���cZ�9/Ƽ�g
                      BJ&W�&$q��e}�2���p��o�KW+ɐ�j;J=k���{��K�9�        ]y�$�.�ה>f�l>D�:f���� ���       �x�@C�������`��3T�8N
�@���U�N%y��6������.徬�o"�0�9>:WA���86Ӻޯ5�ԝb�1�Ƅ�׋����}?�`r�{,�>¯����3d!�K^#ר�T����<���ԍ��x�$���|kG_�u�\�<g(|��á�d�Z�����7�#6f�Ń�1���:����%i���y�I�8��z�7,*հ�����H����X����b/g�{�b�~Ӵ#�Q�V�F��\��aR_u��ά��FUc�5����jP'm�D_%\��[��y#��1�VL;�Yꙫ��=6��9��_��,���g{2�������^�ײ��!�ޚojq�U�
                                                          �鮼y�z|ckn.W@J�����ⰳ[Q�c���WȬ�\�&;8��mdv!�䇎萳G���t�G&#c�� J��U5�`�2{      ��7��7J�^��$�;ϞaHRe*Z%�d��b=��h�1�ґR��*q��d�p�@�6�U�E҄Y7}����厴U��L�@re�:q�cҝ��'Ý��Q�1�q��I�/��<G�z���S���x��
                       �b/���6h�~�]����ko�ml�   EN
T�؍���-����
�
#�}/�ȁ|��e�gZ�Qm)`�ʛ�q���&r��.��>�@����2k��s9�B��䎌���$n�B\o�+���˗��Q��΢�S�׉ �@a���@��2qﰮ�"�5�5��|{��40A��e�d�نL���h�^?x�ت��b(@`�5h�k>��R�Gk�i��Cj[ޭQ�cX�!mbZ\$v64�"�6 �`}��')4�\F�Z3c�����
�,��8M���       載��    *&:2���[�!y]��M3#�;yl������
                                                    �q������즧HP�K�u�   ��b_=�����~��ɨ���"% 
```

Parabéns você instalou e configurou a criptografia “InnoDB Data-at-Rest Encryption” no MySQL com sucesso.
