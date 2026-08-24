# Painel SDR

**Painel de trabalho para time de pré-vendas, montado sobre um CRM existente.**
O SDR abre um lugar só, vê os leads que são dele, o histórico de cada um e os lembretes pendentes —
sem navegar pelo CRM inteiro atrás de informação espalhada.

O painel não duplica a base: ele lê do CRM em tempo real e mantém localmente apenas o que é dele —
usuários, papéis e sessões.

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat-square&logo=nodedotjs&logoColor=white)
![Express](https://img.shields.io/badge/Express-000000?style=flat-square&logo=express&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white)
![Drizzle](https://img.shields.io/badge/Drizzle%20ORM-C5F74F?style=flat-square&logo=drizzle&logoColor=black)
![React](https://img.shields.io/badge/React%2019-61DAFB?style=flat-square&logo=react&logoColor=black)
![Tailwind](https://img.shields.io/badge/Tailwind%20v4-06B6D4?style=flat-square&logo=tailwindcss&logoColor=white)

---

## O que ele faz

**Leads, histórico e lembretes na mesma tela**
Cada lead traz suas atividades e seus lembretes, buscados sob demanda — o SDR não precisa abrir três
telas do CRM para entender onde parou.

**Cada SDR vê o que é dele**
O usuário do painel carrega o identificador correspondente no CRM, e é por esse vínculo que os leads
atribuídos a ele são reconhecidos.

**Gestão de usuários e papéis**
Dois papéis, administrador e SDR. O administrador cadastra e remove usuários e define a quem cada
um corresponde no CRM.

---

## Decisões técnicas que valem nota

**O token do CRM nunca chega ao navegador.**
Todo acesso ao CRM passa por rotas de proxy no servidor, que injetam a credencial no cabeçalho.
Se o front chamasse o CRM direto, a chave estaria no bundle — visível para qualquer usuário do
painel e para qualquer um que abrisse o DevTools.

**Sessão persistida em banco, não em memória.**
O armazenamento de sessão é o MySQL, com expiração de 24 horas, limpeza automática a cada 15
minutos e criação da tabela na primeira execução. Reiniciar o servidor não desloga o time inteiro,
que é o efeito colateral do armazenamento em memória usado por padrão.

**A equipe é derivada dos dados, não cadastrada duas vezes.**
A lista de atendentes sai dos próprios leads: o painel extrai os identificadores distintos de quem
está atribuído e busca cada um, em paralelo. Não existe cadastro de equipe para manter sincronizado
com o CRM — quem aparece nos leads é quem existe.

**Uma ponte explícita entre duas identidades.**
O usuário tem login próprio no painel e um campo separado que aponta para sua identidade no CRM.
São dois sistemas de identidade, e o vínculo entre eles é um dado, não uma suposição por nome ou
e-mail coincidente.

**Preparado para rodar atrás de proxy reverso.**
A aplicação confia no primeiro salto de proxy, para que IP e protocolo originais cheguem corretos
quando servida por trás de um Nginx ou Apache.

**Administrador não se apaga.**
A remoção de usuário recusa explicitamente o próprio identificador da sessão — um clique errado não
deixa o sistema sem administrador.

---

## Como está construído

```
Navegador
   └── painel (React) — nunca fala com o CRM diretamente
        └── servidor Express
             ├── sessão e usuários  → MySQL local
             └── proxy autenticado  → API do CRM
                  ├── /api/leads              lista de leads
                  ├── /api/leads/:id/activities   histórico
                  ├── /api/leads/:id/reminders    lembretes
                  ├── /api/staff/:id          dados do atendente
                  └── /api/team               atendentes distintos dos leads
```

---

## Rodando localmente

```bash
npm install
npm run db:push     # cria as tabelas locais
npm run db:seed     # popula os dados iniciais
npm run dev         # servidor em modo desenvolvimento
```

Produção: `npm run build` e `npm start`. Verificação de tipos com `npm run check`.

---

<details>
<summary><b>Referência — ambiente e estrutura</b></summary>

### Variáveis de ambiente

| Variável | Para que serve |
|---|---|
| `MYSQL_HOST` · `MYSQL_PORT` · `MYSQL_USER` · `MYSQL_PASSWORD` · `MYSQL_DATABASE` | Banco local: usuários e sessões |
| `SESSION_SECRET` | Assinatura do cookie de sessão |
| `CRM_API_URL` | Endereço base da API do CRM |
| `CRM_API_TOKEN` | Credencial de acesso ao CRM |
| `NODE_ENV` | Ambiente de execução |

### Estrutura

```
├── client            SPA em React + Vite
│   └── src/pages     login, painel do SDR, gestão de usuários
├── server
│   ├── auth.ts       sessão, login, papéis e CRUD de usuários
│   ├── routes.ts     rotas de proxy para o CRM
│   ├── db.ts         conexão com o banco local
│   └── static.ts     entrega do build em produção
├── shared/schema.ts  tabelas e validação compartilhadas
└── script            build e seed
```

### Modelo local

Uma tabela: `users`, com login, senha em hash, nome, e-mail, papel (`admin` ou `sdr`) e o
identificador correspondente no CRM. Todo o resto — leads, atividades, lembretes e atendentes —
vive no CRM e é lido sob demanda.

</details>
