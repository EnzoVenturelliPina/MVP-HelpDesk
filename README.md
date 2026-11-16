## Sumário

- [Visão geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Stack](#stack)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Como rodar](#como-rodar)
- [Usuários de demonstração](#usuários-de-demonstração)
- [Fluxo de uso](#fluxo-de-uso)
- [Fluxo de status do chamado](#fluxo-de-status-do-chamado)
- [Perfis e permissões](#perfis-e-permissões)
- [Modo demonstração](#modo-demonstração)
- [Documentação detalhada](#documentação-detalhada)
- [Licença](#licença)

---

## Visão geral

O sistema tem dois perfis:

- **Solicitante** — abre chamados e acompanha os seus. Pode confirmar ou rejeitar a solução proposta pelo técnico.
- **Técnico** — vê a fila de chamados, assume, define a urgência, move no Kanban, comenta internamente e propõe a solução.

A autenticação usa sessão do Django com CSRF. Não há JWT nem token em `localStorage`. O papel do usuário vem do backend (`GET /auth/me/`), nunca de algo editável no frontend.

O frontend também tem um modo demonstração que roda sem o backend, com dados fictícios em `sessionStorage`. Útil para mostrar a interface sem subir o Django.

---

## Arquitetura

O projeto é dividido em dois subprojetos independentes que se comunicam via HTTP:

- **FrontEnd/** — aplicação React com Vite. Consome a API e renderiza a interface. Em modo demonstração, usa um backend simulado em `sessionStorage` e não faz chamadas reais.
- **BackEnd/** — API REST em Django + DRF. Autenticação por sessão e CSRF, regras de negócio em `services.py`, banco PostgreSQL (SQLite em dev).

O frontend envia o cookie de sessão e o header `X-CSRFToken` em todas as requisições autenticadas. O backend é a autoridade final de autorização e validação; o frontend apenas reflete as regras na interface.

---

## Stack

**Frontend:** React 18, Vite 6, React Router 6, Axios, React Hook Form, `@dnd-kit/core`, Lucide React, CSS puro, Vitest + React Testing Library.

**Backend:** Python 3.12, Django 5.2, Django REST Framework 3.16, PostgreSQL (SQLite em dev), `django-filter`, `drf-spectacular`, `django-cors-headers`, `python-dotenv`, pytest + pytest-django, Ruff.

Detalhes completos em cada README específico.

---

## Estrutura do repositório

```text
.
├── FrontEnd/     # Aplicação React + Vite
├── BackEnd/      # API Django REST Framework
└── README.md
Como rodar
Pré-requisitos: Node.js 18+, npm 9+, Python 3.12+. PostgreSQL é opcional — SQLite funciona para rodar localmente.

Backend
bash
cd BackEnd
python -m venv .venv
source .venv/bin/activate          # Linux/macOS
# .\.venv\Scripts\Activate.ps1     # Windows PowerShell

pip install -r requirements.txt
cp .env.example .env
python manage.py migrate
python manage.py create_technician --username carlos.lima --email carlos.lima@empresa.com
python manage.py seed_demo
python manage.py runserver
API em 
http://localhost:8000/api/
. Swagger em http://localhost:8000/api/docs/.

Frontend
bash
cd FrontEnd
npm install
cp .env.example .env    # VITE_DEMO_MODE=false para usar o backend real
npm run dev
Aplicação em http://localhost:5173.

Só o frontend (modo demonstração)
Se não quiser subir o Django, mantenha VITE_DEMO_MODE=true no .env do frontend. A interface roda com dados fictícios em memória e mostra o selo "Modo demonstração" no cabeçalho.

Usuários de demonstração
Criados pelo seed_demo no backend (ou disponíveis no modo demonstração do frontend):

Perfil	Usuário	Senha
Solicitante	ana.souza / ana.souza@demo.local	demo1234
Solicitante	joao.pereira	
demo1234
Técnico	
carlos.lima
demo1234
Técnico	marina.alves	demo1234
O login aceita username ou e-mail no campo username.

Fluxo de uso
Roteiro para entender o sistema de ponta a ponta:

Entre como Ana (solicitante), abra um chamado e acompanhe em "Meus chamados".

Saia e entre como Carlos (técnico). Assuma o chamado pelo Kanban definindo a urgência.

Ainda como Carlos, registre um comentário interno e use "Solicitar confirmação" com a solução proposta.

Volte como Ana, abra o chamado pendente e confirme a resolução. O status vai para RESOLVIDO.

Repita o fluxo rejeitando a solução para ver o chamado voltar para EM_ATENDIMENTO, mantendo o técnico responsável.

Fluxo de status do chamado
text
ABERTO
  └── assign (urgência obrigatória) ──▶ EM_ATENDIMENTO
                                           ├── transition ──▶ AGUARDANDO_SOLICITANTE
                                           ├── resolve ─────▶ AGUARDANDO_SOLICITANTE
                                           └── transition ◀── AGUARDANDO_SOLICITANTE

AGUARDANDO_SOLICITANTE
  ├── confirm-resolution {resolved: true}  ──▶ RESOLVIDO (terminal)
  └── confirm-resolution {resolved: false} ──▶ EM_ATENDIMENTO
Algumas regras que valem a pena ter em mente:

ABERTO só sai por assign, que define responsável, urgência e status de uma vez.

Só o técnico responsável altera o status operacional.

resolve
 exige uma solução com pelo menos 10 caracteres e move para 
AGUARDANDO_SOLICITANTE
.

Só o solicitante proprietário confirma ou rejeita, e só em 
AGUARDANDO_SOLICITANTE
.

Confirmação positiva preenche resolved_at. Negativa limpa a solução e mantém o técnico.

RESOLVIDO é terminal no MVP.

Toda operação válida gera evento de auditoria na mesma transação.

Conflitos de estado ou concorrência retornam 409 Conflict.

Perfis e permissões
Ação	Solicitante	Técnico responsável	Outro técnico
Abrir chamado	Sim	Sim	Sim
Ver próprios chamados	Sim	—	—
Ver fila de chamados	Não	Sim	Sim
Assumir chamado ABERTO	Não	Sim	Sim
Definir urgência na assunção	Não	Sim	Sim
Mover no Kanban / alterar status	Não	Sim	Não
Solicitar confirmação da resolução	Não	Sim	Não
Confirmar/rejeitar a resolução	Somente o proprietário	Não	Não
Comentários internos	Não	Sim	Sim
Histórico de auditoria	Próprios chamados (eventos seguros)	Sim	Sim
O frontend reflete essas regras em src/utils/ticketPermissions.js, mas a autorização real é sempre do backend. Permissões de UI são só conveniência de experiência.

Modo demonstração
Com VITE_DEMO_MODE
=true
 no frontend:

Os serviços (authService, 
ticketService
, etc.) delegam para src/services/demo/demoBackend
.js
.

Os dados ficam em memória e em 
sessionStorage
 (chaves helpdesk.demo.db.v1 e helpdesk.demo.session.v1).

A sessão dura enquanto a aba estiver aberta. Limpar o sessionStorage restaura os dados originais.

Nenhuma chamada à API real é feita.

Serve para demonstrar a interface sem subir o backend, testar fluxos de UI isoladamente e desenvolver frontend sem depender do Django.

Documentação detalhada
Frontend: FrontEnd/README.md — stack, estrutura de pastas, rotas, contrato da API, testes, acessibilidade e limitações.

Backend: BackEnd/README.md — arquitetura, endpoints, fluxo de status, permissões, filtros, testes e segurança.

Licença
Projeto pessoal para fins de demonstração de competências Full Stack. Sem licença definida.