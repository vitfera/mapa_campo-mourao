# Design: pgAdmin em Produção

## Objetivo

Adicionar um serviço `pgadmin` ao ambiente de produção do projeto para permitir administração do banco PostgreSQL/PostGIS via navegador, com acesso restrito ao ambiente já protegido por VPN.

## Escopo

- Adicionar um novo serviço `pgadmin` ao arquivo `docker-compose.yml`
- Configurar credenciais iniciais e porta pública via variáveis do arquivo `.env`
- Persistir dados do `pgAdmin` em `./docker-data/pgadmin`
- Permitir acesso ao banco `db` já existente no stack Docker
- Documentar as variáveis novas no arquivo `.env_sample`

## Fora de Escopo

- Publicação do `pgAdmin` atrás do `nginx`
- Criação de domínio próprio ou TLS dedicado para o `pgAdmin`
- Alterações na configuração do PostgreSQL
- Cadastro automático de servidores dentro do `pgAdmin`

## Abordagem Escolhida

O `pgAdmin` será adicionado como um serviço persistente no `docker-compose.yml` principal, subindo junto com o stack de produção. O container usará a imagem oficial `dpage/pgadmin4`, terá volume dedicado para persistência e publicará uma porta alta configurável do host para a porta `80` interna.

As credenciais iniciais serão lidas do `.env` usando:

- `PGADMIN_DEFAULT_EMAIL`
- `PGADMIN_DEFAULT_PASSWORD`
- `PGADMIN_PORT`

## Alternativas Consideradas

### Serviço opcional

Manter o `pgAdmin` configurado, mas sem subir por padrão. Isso reduziria exposição permanente, mas adicionaria atrito operacional e não foi a opção escolhida.

### Compose separado

Isolar o `pgAdmin` em um compose específico de administração. Isso aumentaria a separação, mas complicaria a manutenção sem trazer benefício proporcional para o cenário atual.

## Arquitetura

O novo serviço `pgadmin` ficará no mesmo stack Docker do serviço `db`, usando a rede padrão do compose. O acesso ao banco será feito pelo hostname interno `db`, sem necessidade de expor portas adicionais do PostgreSQL além das que já existirem no ambiente.

O acesso web ao `pgAdmin` será feito por:

`http://IP_DO_SERVIDOR:PGADMIN_PORT`

O uso pressupõe que o ambiente já esteja acessível somente por VPN, conforme definido pelo time.

## Persistência

Será criado um volume bind mount:

- `./docker-data/pgadmin:/var/lib/pgadmin`

Isso preserva preferências, conexões salvas e demais dados do `pgAdmin` entre reinicializações e recriações do container.

## Segurança

- O serviço ficará protegido principalmente pelo isolamento de rede já existente via VPN
- O acesso exigirá autenticação no `pgAdmin` com credenciais definidas no `.env`
- A porta será configurável para uso de valor alto não padrão
- A senha inicial deve ser forte e exclusiva do ambiente

## Fluxo Operacional

1. Definir `PGADMIN_DEFAULT_EMAIL`, `PGADMIN_DEFAULT_PASSWORD` e `PGADMIN_PORT` no `.env`
2. Subir o stack de produção normalmente
3. Acessar o `pgAdmin` pela porta configurada dentro da VPN
4. Cadastrar o servidor PostgreSQL usando host `db`, porta `5432` e as credenciais já usadas pelo serviço de banco

## Impacto em Arquivos

- `docker-compose.yml`: incluir o serviço `pgadmin`
- `.env_sample`: documentar as novas variáveis de ambiente

## Testes e Validação

- Validar a sintaxe do compose após a alteração
- Subir o serviço e verificar se o container inicia sem erro
- Confirmar acesso web com as credenciais definidas no `.env`
- Confirmar conexão manual do `pgAdmin` ao host `db`

## Riscos Conhecidos

- Exposição desnecessária caso a porta seja liberada fora da VPN
- Uso de senha fraca no `PGADMIN_DEFAULT_PASSWORD`
- Acúmulo de dados persistidos no diretório `docker-data/pgadmin` sem política de backup/limpeza
