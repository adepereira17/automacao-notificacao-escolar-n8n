# Documentação do Projeto

## 1. Visão geral

Este projeto tem como objetivo desenvolver uma automação para envio de notificações escolares utilizando o **n8n**, integrado ao banco de dados **MySQL** do sistema de gestão escolar.

A automação identifica novas avaliações e atividades cadastradas, consulta os alunos relacionados às respectivas turmas e realiza o envio de notificações por e-mail e WhatsApp.

A solução busca reduzir atividades manuais da Coordenação Pedagógica e melhorar a comunicação com alunos e responsáveis.

---

## 2. Problema

Em um cenário tradicional, a comunicação de uma nova avaliação pode depender de processos manuais.

A equipe responsável precisa:

1. Identificar novas avaliações;
2. Consultar a turma envolvida;
3. Identificar os alunos;
4. Verificar os responsáveis;
5. Selecionar os contatos;
6. Preparar a comunicação;
7. Enviar as mensagens;
8. Controlar quem já recebeu a informação.

Esse processo pode consumir tempo e aumentar a possibilidade de erros ou esquecimentos.

---

## 3. Objetivo

Automatizar o processo de comunicação de avaliações e atividades escolares.

A automação deve:

* Consultar periodicamente o MySQL;
* Identificar novas atividades;
* Localizar os alunos da turma;
* Identificar responsáveis de alunos menores de idade;
* Criar os destinatários;
* Enviar notificações;
* Evitar notificações duplicadas;
* Registrar o resultado dos envios;
* Permitir tratamento de erros.

---

## 4. Público

A solução foi pensada principalmente para a:

**Coordenação Pedagógica**

Também pode beneficiar:

* Secretaria;
* Professores;
* Alunos;
* Responsáveis.

---

## 5. Ferramentas

### n8n

Responsável pela execução e orquestração da automação.

### MySQL

Banco de dados utilizado pelo sistema de gestão escolar.

O MySQL fornece os dados necessários para o funcionamento do workflow, como:

* Atividades;
* Turmas;
* Alunos;
* Responsáveis;
* Dados de contato.

### E-mail

Utilizado como um dos canais de comunicação.

### WhatsApp

Utilizado como canal complementar de comunicação por meio de uma API ou serviço compatível.

### Git e GitHub

Utilizados para versionamento e documentação do projeto.

---

## 6. Fluxo de negócio

O funcionamento geral da automação ocorre da seguinte forma:

1. O n8n inicia a execução automaticamente em intervalos definidos;
2. O workflow consulta o banco de dados MySQL;
3. São identificadas novas atividades escolares;
4. Para cada atividade, a turma correspondente é identificada;
5. O MySQL é consultado para obter os alunos da turma;
6. Os dados dos responsáveis são recuperados;
7. A idade dos alunos é analisada;
8. São definidos os destinatários;
9. Cada destinatário é processado individualmente;
10. O workflow verifica se a notificação já foi enviada;
11. Caso ainda não tenha sido enviada, realiza o envio;
12. O resultado é registrado no MySQL.

---

## 7. Regra de negócio

A principal regra do projeto está relacionada à idade do aluno.

### Alunos menores de 18 anos

Devem receber a notificação:

* O próprio aluno;
* Seu responsável cadastrado.

### Alunos com 18 anos ou mais

Devem receber a notificação:

* Somente o próprio aluno.

Essa regra é aplicada automaticamente pelo workflow.

---

## 8. Dados necessários

Para que a automação funcione, o banco de dados MySQL deve disponibilizar informações semelhantes às seguintes.

### Atividade

* ID da atividade;
* Data;
* Tipo;
* Descrição;
* Disciplina;
* Turma.

### Aluno

* ID;
* Nome;
* Data de nascimento;
* E-mail;
* Telefone;
* Turma.

### Responsável

* ID;
* Nome;
* E-mail;
* Telefone;
* Relação com o aluno.

Os nomes das tabelas e campos devem ser ajustados de acordo com o banco MySQL real utilizado pela instituição.

---

## 9. Estratégia de execução

O workflow utiliza um **Schedule Trigger** para executar automaticamente em intervalos definidos.

Como exemplo, a automação pode ser executada a cada:

**5 minutos**

A cada execução, o n8n consulta o MySQL procurando novas atividades que ainda não tenham sido processadas.

Exemplo de consulta:

```sql
SELECT
    id,
    data_atividade,
    tipo,
    descricao,
    materia_id,
    turma_id
FROM atividades_escolares
WHERE data_atividade >= CURRENT_DATE
  AND notificacao_processada = 0;
```

A consulta é apenas um exemplo e deve ser adaptada ao schema real do MySQL.

---

## 10. Controle de duplicidade

Uma das principais preocupações da automação é evitar o envio repetido de uma mesma notificação.

Para isso, deve existir uma tabela de controle no MySQL.

A identificação de uma notificação considera:

* ID da atividade;
* ID do aluno;
* Tipo do destinatário;
* Canal utilizado.

Exemplo de consulta:

```sql
SELECT id
FROM notificacoes_atividades
WHERE atividade_id = ?
  AND aluno_id = ?
  AND destinatario_tipo = ?
  AND canal = ?;
```

Se existir um registro correspondente, o envio não deve ser realizado novamente.

---

## 11. Tratamento de erros

A automação deve considerar que diferentes destinatários podem apresentar problemas individuais.

Exemplos:

* E-mail inválido;
* Telefone ausente;
* API do WhatsApp indisponível;
* Falha de conexão com o MySQL;
* Dados incompletos;
* Erro temporário no serviço de comunicação.

Um erro em um destinatário não deve interromper o processamento dos demais.

Sempre que possível, o erro deve ser registrado para permitir análise e reprocessamento posterior.

---

## 12. Comunicação

### E-mail

Exemplo de mensagem:

**Assunto: Nova atividade escolar**

Olá, [Nome].

Uma nova atividade foi cadastrada para sua turma.

**Disciplina:** [Disciplina]

**Atividade:** [Descrição]

**Data:** [Data]

Acesse o sistema de gestão escolar para consultar mais informações.

Atenciosamente,
Coordenação Pedagógica

---

### WhatsApp

Exemplo:

Olá, [Nome]! 📚

Foi cadastrada uma nova atividade para sua turma.

📖 Disciplina: [Disciplina]
📝 Atividade: [Descrição]
📅 Data: [Data]

Acesse o sistema de gestão escolar para consultar mais informações.

---

## 13. Segurança

As credenciais utilizadas pelo workflow não devem ser armazenadas diretamente no código.

As conexões devem utilizar o sistema de credenciais do n8n.

Especialmente para o MySQL, devem ser protegidos:

* Host;
* Porta;
* Usuário;
* Senha;
* Banco de dados.

As credenciais não devem ser versionadas no GitHub.

---

## 14. LGPD

O projeto trabalha com dados pessoais de alunos e responsáveis.

Entre os dados potencialmente utilizados estão:

* Nome;
* Data de nascimento;
* E-mail;
* Telefone.

Por isso, a automação deve considerar princípios de segurança e proteção de dados.

Boas práticas:

* Utilizar somente os dados necessários;
* Restringir o acesso ao banco MySQL;
* Proteger credenciais;
* Evitar armazenar informações desnecessárias;
* Controlar o acesso aos workflows;
* Não publicar dados reais no GitHub;
* Utilizar dados fictícios durante demonstrações e testes.

---

## 15. Monitoramento

O workflow deve permitir acompanhar:

* Quantidade de atividades encontradas;
* Quantidade de alunos processados;
* Quantidade de notificações enviadas;
* Quantidade de falhas;
* Canal utilizado;
* Data e horário do envio.

O histórico pode ser armazenado em uma tabela específica no MySQL.

---

## 16. Benefícios

A automação proporciona:

* Redução do trabalho manual;
* Maior agilidade na comunicação;
* Redução de esquecimentos;
* Controle dos envios;
* Histórico das notificações;
* Processamento automático;
* Facilidade de manutenção;
* Possibilidade de expansão para outros tipos de comunicação.

---

## 17. Resultado esperado

Ao final da implementação, espera-se que uma atividade cadastrada no sistema de gestão escolar seja automaticamente identificada pelo n8n.

A partir disso, o workflow deve:

1. Consultar o MySQL;
2. Identificar a atividade;
3. Localizar a turma;
4. Localizar os alunos;
5. Identificar os responsáveis;
6. Aplicar a regra de idade;
7. Verificar duplicidade;
8. Enviar e-mail;
9. Enviar WhatsApp;
10. Registrar o resultado.

Dessa forma, a Coordenação Pedagógica não precisa realizar manualmente todo o processo de comunicação.

---

## 18. Estrutura de controle

Exemplo de tabela MySQL:

```sql
CREATE TABLE notificacoes_atividades (
    id INT PRIMARY KEY AUTO_INCREMENT,
    atividade_id INT NOT NULL,
    aluno_id INT NOT NULL,
    destinatario_tipo VARCHAR(20) NOT NULL,
    canal VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL,
    enviado_em DATETIME,
    mensagem_erro TEXT
);
```

Para uma implementação real, recomenda-se também avaliar a criação de uma restrição `UNIQUE` para reforçar o controle de duplicidade:

```sql
ALTER TABLE notificacoes_atividades
ADD CONSTRAINT uq_notificacao
UNIQUE (
    atividade_id,
    aluno_id,
    destinatario_tipo,
    canal
);
```

---

## 19. Considerações finais

O projeto demonstra como o n8n pode ser utilizado para integrar um sistema de gestão escolar baseado em MySQL a diferentes canais de comunicação.

A arquitetura permite que o processo seja automatizado sem a necessidade de intervenção manual a cada nova atividade cadastrada.

A estrutura também pode ser expandida futuramente para outros eventos, como:

* Reuniões;
* Comunicados;
* Eventos;
* Prazos;
* Entrega de trabalhos;
* Alterações de horários;
* Avisos administrativos.
