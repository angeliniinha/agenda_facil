# Agenda Fácil

Projeto integrador de Banco de Dados e Programação Mobile/Web. Aplicação responsiva para agendamento de serviços de **um profissional**, com acesso de cliente e administrador.

## Funcionalidades

- Cadastro e login de clientes; administrador criado pelas variáveis de ambiente.
- Cadastro, edição e desativação de serviços.
- Disponibilização e bloqueio de horários futuros.
- Reserva de atendimento, consulta do histórico, cancelamento e conclusão pelo profissional.
- Proteção contra reservas simultâneas, inclusive no banco de dados.
- Preço preservado no agendamento mesmo depois de alterar o preço do serviço.

Todos os atendimentos duram **60 minutos** e começam em horas inteiras. Datas são armazenadas com fuso e exibidas no fuso do dispositivo. O cliente só acessa seus agendamentos. O administrador acessa a agenda completa. Cancelamentos são permitidos antes do início; conclusão, apenas após o fim. Não há pagamentos, mensagens, recuperação de senha ou múltiplos profissionais nesta versão.

## Tecnologias e estrutura

React + Vite + CSS no frontend; Node.js + Express na API; PostgreSQL no banco. Autenticação JWT de 8 horas e senhas com hash bcrypt. O token fica no `sessionStorage` da aba. As consultas SQL usam parâmetros.

```text
src/                        Interface React e estilos responsivos
server/index.js             API, autenticação e regras transacionais
server/schema.sql           Tabelas, chaves e restrições
server/rules.js             Validação das entradas
server/*.test.js            Testes unitários e de integração
.github/workflows/ci.yml    Verificação automática no GitHub
compose.yaml                Aplicação e PostgreSQL via Docker
```

## Executar com Docker — recomendado

Instale Docker Desktop com suporte a containers Linux. Na pasta do projeto:

```powershell
Copy-Item .env.example .env
```

No Linux/macOS, use `cp .env.example .env`.

Edite `.env`: escolha `ADMIN_EMAIL`, uma senha `ADMIN_PASSWORD` de pelo menos 8 caracteres e uma `JWT_SECRET` aleatória de pelo menos 32 caracteres. Não publique `.env`. Se tiver Node instalado, gere uma chave com:

```sh
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

Inicie:

```sh
docker compose up --build -d
```

Abra **http://localhost:3000**. O banco e o administrador são criados automaticamente. Entre com o e-mail e senha definidos no `.env`. Cadastre um serviço e disponibilize horários futuros. Em outra janela do navegador, crie uma conta de cliente e faça uma reserva.

Para parar: `docker compose down`. Os dados ficam no volume `agenda_data`. Alterar `ADMIN_PASSWORD` no `.env` **não muda a senha de um administrador já criado**; a configuração inicial só cria contas ausentes. As credenciais do PostgreSQL no Compose são para desenvolvimento local. A aplicação e o banco estão publicados apenas no endereço local do computador.

## Executar em desenvolvimento

Requer Node.js 22 ou superior com npm e PostgreSQL 16 (ou Docker para iniciar somente o banco).

1. Copie e configure `.env` conforme acima.
2. Inicie o banco: `docker compose up db -d`. Se usar PostgreSQL instalado, crie um banco vazio e ajuste `DATABASE_URL`.
3. Instale dependências e inicie:

```sh
npm ci
npm run dev
```

Abra **http://localhost:5173**. O Vite encaminha `/api` para a API na porta 3000.

Para compilar e servir pela API:

```sh
npm run build
npm start
```

## Testes

```sh
npm test
npm run build
```

O teste de integração exige API e PostgreSQL em execução. Ele **cria dados de teste**, então use um banco exclusivo para testes. Configure as mesmas credenciais de administrador do servidor e a URL da API. No PowerShell:

```powershell
$env:TEST_BASE_URL = 'http://localhost:3000'
$env:ADMIN_EMAIL = 'seu-admin-de-teste@example.com'
$env:ADMIN_PASSWORD = 'SuaSenhaDeTeste123!'
npm run test:integration
```

Sem `TEST_BASE_URL`, o teste de integração é ignorado. No GitHub Actions, o workflow cria um PostgreSQL isolado e executa a compilação e todos os testes. A integração verifica disputa simultânea pelo mesmo horário, acesso de outro cliente, bloqueio, cancelamento, preservação do preço e serviço inativo.

## Modelo de dados

| Tabela | Chaves e dados | Relações |
|---|---|---|
| users | id (PK), name, email (único), password_hash, role | 1:N appointments |
| services | id (PK), name, description, price, active | 1:N appointments |
| slots | id (PK), starts_at (único), ends_at, blocked | 1:N appointments no histórico |
| appointments | id (PK), user_id/service_id/slot_id (FK), status, price_at_booking, created_at | N:1 com cada tabela acima |

Um índice único parcial permite apenas um agendamento não cancelado por horário. A reserva e o bloqueio de um horário usam transações com bloqueio de linha. Os horários de 60 minutos com início em horas inteiras evitam sobreposição. Desativar um serviço preserva seu histórico.

## Rotas principais da API

Todas as rotas abaixo usam `/api`. Rotas protegidas exigem `Authorization: Bearer <token>`.

| Método | Rota | Permissão |
|---|---|---|
| POST | /auth/register | Pública: name, email, password |
| POST | /auth/login | Pública: email, password |
| GET | /me | Autenticado |
| GET | /services | Autenticado; cliente vê ativos |
| POST / PUT | /services / /services/:id | Administrador |
| GET | /slots | Autenticado; cliente vê livres |
| POST | /slots | Administrador; starts_at em ISO com fuso |
| PATCH | /slots/:id | Administrador; blocked booleano |
| GET / POST | /appointments | Autenticado; criação usa service_id e slot_id |
| PATCH | /appointments/:id | Dono ou administrador; status cancelled/completed |

## Colocar no GitHub

Crie um repositório vazio no GitHub. Extraia o ZIP e abra o terminal **dentro da pasta `agenda-facil`**, onde está este README. Execute, substituindo a URL:

```sh
git init
git add .
git commit -m "Projeto inicial Agenda Fácil"
git branch -M main
git remote add origin https://github.com/SEU-USUARIO/agenda-facil.git
git push -u origin main
```

O `.gitignore` exclui `.env`, `node_modules` e `dist`. Confira os arquivos antes de enviar. O código inclui `.env.example`, sem credenciais reais. No GitHub, acompanhe a aba **Actions** para ver o resultado dos testes.

**GitHub armazena o código; GitHub Pages não executa esta API nem o PostgreSQL.** Para demonstrar, rode localmente. Para publicar o sistema na internet, será necessário hospedar o servidor Node e o PostgreSQL, configurar HTTPS, credenciais próprias e backups.

## Roteiro para a banca

1. Mostrar os serviços e um horário disponível no painel do profissional.
2. Criar uma reserva como cliente e mostrar o agendamento na agenda do profissional.
3. Mostrar que o horário reservado não aparece disponível para outro cliente.
4. Cancelar a reserva e mostrar o horário disponível novamente.
5. Apresentar as tabelas, as chaves estrangeiras e o índice único parcial.
