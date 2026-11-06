# SGRH-SUAS Beruri

Sistema de Gestão de Recursos Humanos do SUAS para a Secretaria Municipal de
Assistência Social de Beruri/AM.

## Requisitos locais

- Node.js 20.9 ou superior.
- PostgreSQL local.
- Visual Studio Code.

## Preparar o ambiente

No PowerShell, na pasta do projeto:

```powershell
Copy-Item .env.example .env
```

Edite `DATABASE_URL` no `.env` com o usuário, a senha e o nome do banco
PostgreSQL local. O arquivo `.env` é ignorado pelo Git e não deve ser compartilhado.

Depois de iniciar o serviço PostgreSQL e criar o banco `sgrh_suas`, execute:

No PowerShell do Windows, use `npm.cmd` para evitar bloqueio de política de
execução ao invocar `npm.ps1`:

```powershell
npm.cmd install
npm.cmd run db:validate
npm.cmd run db:generate
npm.cmd run db:migrate
npm.cmd run auth:create-admin
npm.cmd run dev
```

O comando `auth:create-admin` cria o primeiro administrador e solicita a senha
sem exibi-la no terminal. Use uma senha forte, não reutilizada, e guarde-a em
local seguro. Não existem contas ou senhas padrão e o cadastro público está
desabilitado.

Para criar uma segunda conta individual com perfil de Gestor de RH, execute
`npm.cmd run auth:create-hr-user`. A senha também é solicitada sem aparecer no
terminal.

Abra <http://localhost:3000> e entre com a conta criada. Para consultar o banco
durante o desenvolvimento:

```powershell
npm.cmd run db:studio
```

## Estrutura inicial

- `src/app`: rotas e interface Next.js com React, TypeScript e Tailwind CSS.
- `src/lib/prisma.ts`: cliente Prisma compartilhado com adaptador PostgreSQL.
- `src/lib/session.ts`: sessões de banco com cookie HTTP-only e autorização por papel.
- `src/lib/password-hash.mjs`: hash de senha com scrypt e salt aleatório.
- `scripts/create-admin.mjs`: criação local do primeiro administrador, sem senha ecoada.
- `tests`: testes unitários de hash e verificação de senha.
- `prisma/schema.prisma`: modelos de servidores, equipamentos, lotações,
  arquivos, usuários, permissões e auditoria.
- `prisma/migrations`: migrations versionadas do banco de dados.
- `.env.example`: nomes e formato das configurações locais; não contém credenciais.
- `.private-storage`: destino de arquivos privados, fora de `public/` e ignorado
  pelo Git.

As fotos de servidores são gravadas em armazenamento privado, com metadados no
banco e acesso autenticado. No desenvolvimento local, `PRIVATE_STORAGE_PATH`
aponta para um diretório privado em disco; o padrão é `.private-storage`, fora
de `public/`. Em produção, configure um bucket privado Cloudflare R2 com
`R2_ACCOUNT_ID`, `R2_BUCKET_NAME`, `R2_ACCESS_KEY_ID` e `R2_SECRET_ACCESS_KEY`.
As quatro variáveis são obrigatórias em produção; o sistema não usa arquivos
locais como fallback em ambiente publicado. Crie uma chave limitada ao bucket e
cadastre os valores como variáveis privadas no provedor de hospedagem. O cadastro
aceita JPEG, PNG e WebP de até 1 MB. Não coloque arquivos pessoais em `public/`.

## Publicação na Cloudflare

O projeto está configurado para Cloudflare Workers usando vinext. Esse caminho
está em beta; valide o preview e os fluxos de autenticação, banco e fotos antes
de usar dados reais. O `cloudflare.config.ts` usa `nodejs_compat` e o script
`npm.cmd run build:vinext` gera o Worker.

O limite de CPU está configurado para 30 segundos, necessário para o hash de
senhas scrypt. Verifique no painel da Cloudflare se o plano do Worker permite
esse limite antes de publicar.

1. Crie um banco PostgreSQL acessível a partir da Cloudflare. Configure o acesso
   de produção por um serviço Cloudflare Hyperdrive ou por um driver compatível
   com Workers; `DATABASE_URL` deve ser o endpoint de conexão de produção.
2. Crie um bucket R2 privado e uma chave S3 com acesso limitado a esse bucket.
3. Na Cloudflare, cadastre como secrets `DATABASE_URL`, `R2_ACCOUNT_ID`,
   `R2_BUCKET_NAME`, `R2_ACCESS_KEY_ID` e `R2_SECRET_ACCESS_KEY`. Nunca use o
   prefixo `NEXT_PUBLIC_` para credenciais.
4. Conecte um repositório privado e configure o build como `npm run
   db:generate && npm run build:vinext`, ou use `npm.cmd run deploy:vinext`
   após autenticar o Wrangler.
5. Antes de liberar o acesso, aplique as migrações no banco de produção com
   `npm.cmd run db:deploy` e crie as contas com `npm.cmd run auth:create-admin`
   e `npm.cmd run auth:create-hr-user`, apontando para o banco correto. Os
   comandos pedem as senhas sem exibi-las.

Para desenvolver com o runtime de Workers, copie `.dev.vars.example` para
`.dev.vars`, preencha os valores de um ambiente de teste e execute
`npm.cmd run dev:vinext`. O arquivo `.dev.vars` é local e ignorado pelo Git.
Não publique o sistema nem cadastre dados reais antes de validar o acesso ao
PostgreSQL no runtime Workers, o envio/leitura de fotos no R2 e a restauração
dos backups.

## Estado da implementação

A Etapa 1 preparou o projeto, as dependências e o modelo relacional. A Etapa 2
adicionou login, sessões, papéis, permissões e auditoria. As telas protegidas
permitem cadastrar, consultar e editar servidores; cadastrar equipamentos;
alterar situação funcional; consultar histórico e registrar transferências.
Fotos ficam privadas e só são servidas a perfis com permissão de leitura de dados
sensíveis. Relatórios, documentos funcionais e administração de usuários ainda
precisam ser implementados.
