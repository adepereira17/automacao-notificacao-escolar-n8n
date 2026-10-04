# Automação de Notificações Escolares com n8n

Automação desenvolvida com **n8n** para enviar notificações de avaliações e atividades escolares aos alunos e responsáveis, utilizando dados armazenados em um banco de dados **MySQL**.

O objetivo é reduzir o trabalho manual da Coordenação Pedagógica e garantir que as informações sobre provas e atividades sejam comunicadas de forma rápida e organizada.

## Objetivo

Automatizar o processo de comunicação de novas avaliações e atividades cadastradas no sistema de gestão escolar.

O workflow consulta periodicamente o banco de dados MySQL, identifica novas atividades, verifica os alunos envolvidos e envia as notificações pelos canais disponíveis.

## Fluxo da automação

```text
┌─────────────────────┐
│ Schedule Trigger    │
│ Execução periódica  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ MySQL               │
│ Buscar atividades   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Existem atividades? │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ MySQL               │
│ Buscar alunos e     │
│ responsáveis        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Code                │
│ Montar destinatários│
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Split Out           │
│ Um destinatário     │
│ por item             │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Verificar duplicidade│
└──────────┬──────────┘
           │
           ▼
     ┌─────┴─────┐
     │           │
     ▼           ▼
┌─────────┐ ┌───────────┐
│ E-mail  │ │ WhatsApp  │
└─────────┘ └───────────┘
     │           │
     └─────┬─────┘
           ▼
┌─────────────────────┐
│ MySQL               │
│ Registrar resultado │
└─────────────────────┘
```

## Regra de destinatários

A automação considera a idade do aluno para definir quem deve receber a comunicação.

### Aluno menor de idade

A notificação será enviada para:

* Aluno
* Responsável

### Aluno maior ou igual a 18 anos

A notificação será enviada somente para:

* Aluno

A regra é aplicada automaticamente durante a execução do workflow.

## Banco de dados

O projeto utiliza **MySQL** como banco de dados do sistema de gestão escolar.

O n8n realiza consultas diretamente no MySQL para:

* Identificar novas atividades;
* Consultar alunos;
* Consultar responsáveis;
* Recuperar informações de contato;
* Verificar notificações já enviadas;
* Registrar o resultado das notificações.

Os nomes das tabelas apresentados neste projeto são exemplos e devem ser adaptados à estrutura real do banco de dados MySQL utilizado pela instituição.

## Canais de comunicação

A automação foi planejada para utilizar dois canais principais:

* E-mail
* WhatsApp

O envio de WhatsApp pode ser realizado por meio de uma API ou serviço externo compatível.

## Controle de duplicidade

O workflow possui uma camada de controle para impedir que uma mesma notificação seja enviada repetidamente.

A combinação utilizada para identificar uma notificação considera:

* Atividade;
* Aluno;
* Tipo de destinatário;
* Canal.

O histórico é armazenado no MySQL.

## Tratamento de erros

A automação considera possíveis falhas durante o processamento, como:

* E-mail inválido;
* Telefone inexistente;
* Falha na API do WhatsApp;
* Falha na conexão com o MySQL;
* Dados incompletos;
* Erro no processamento de um destinatário.

A ocorrência de erro em um destinatário não deve impedir o processamento dos demais.

## Documentação

### Documentação do projeto

[docs/projeto.md](docs/projeto.md)

Apresenta o problema, objetivo, regras de negócio, segurança, LGPD e funcionamento geral da solução.

### Documentação do workflow

[workflow/README.md](workflow/README.md)

Apresenta a estrutura técnica do workflow no n8n, seus nós, consultas MySQL e estratégias de controle.

## Tecnologias utilizadas

* n8n
* MySQL
* SQL
* JavaScript
* E-mail
* WhatsApp API
* Git
* GitHub

## Status

Projeto desenvolvido como parte de uma atividade prática de automação utilizando n8n.

A estrutura apresentada é um modelo de implementação e deve ser adaptada ao banco de dados MySQL, sistema de gestão escolar e serviços de comunicação utilizados no ambiente real.

## Projeto

**Automação de Notificações Escolares**

Tecnologia principal: **n8n + MySQL**
