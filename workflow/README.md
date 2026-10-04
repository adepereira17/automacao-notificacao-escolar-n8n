# Workflow n8n

Este documento apresenta a estrutura técnica do workflow responsável pela automação das notificações escolares.

A automação utiliza o **n8n** para consultar o banco de dados **MySQL** do sistema de gestão escolar, processar os destinatários e realizar os envios por e-mail e WhatsApp.

---

## 1. Arquitetura

```text
Schedule Trigger
       │
       ▼
MySQL - Buscar Novas Atividades
       │
       ▼
Existem Atividades?
       │
       ▼
MySQL - Buscar Alunos e Responsáveis
       │
       ▼
Code - Montar Destinatários
       │
       ▼
Split Out
       │
       ▼
MySQL - Verificar Duplicidade
       │
       ▼
Controle de E-mail
       │
       ├──────► Enviar E-mail
       │
       ▼
Controle de WhatsApp
       │
       ├──────► Enviar WhatsApp
       │
       ▼
MySQL - Registrar Resultado
```

---

# 2. Nós do workflow

## 2.1 Schedule Trigger

Responsável por iniciar automaticamente a execução.

Exemplo:

```text
Intervalo: 5 minutos
```

O intervalo pode ser ajustado de acordo com a necessidade da instituição.

---

## 2.2 MySQL - Buscar Novas Atividades

Responsável por consultar o banco MySQL e identificar atividades que precisam ser processadas.

Exemplo:

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

### Observação

Os nomes `atividades_escolares`, `materia_id`, `turma_id` e `notificacao_processada` são exemplos.

Eles devem ser substituídos pelos nomes existentes no banco MySQL real.

---

# 3. Existem Atividades?

Um nó `IF` verifica se a consulta retornou alguma atividade.

### Se existir atividade

O fluxo continua para a consulta de alunos.

### Se não existir

O workflow é encerrado até a próxima execução automática.

---

# 4. MySQL - Buscar Alunos e Responsáveis

Depois de identificar uma atividade, o workflow consulta o MySQL para localizar os alunos pertencentes à turma.

Exemplo:

```sql
SELECT
    a.id AS aluno_id,
    a.nome AS aluno,
    a.data_nascimento,
    a.email AS email_aluno,
    a.telefone AS telefone_aluno,
    r.id AS responsavel_id,
    r.nome AS responsavel,
    r.email AS email_responsavel,
    r.telefone AS telefone_responsavel
FROM alunos a
LEFT JOIN responsaveis r
    ON r.id = a.responsavel_id
WHERE a.turma_id = ?;
```

O parâmetro `?` deve receber o ID da turma relacionado à atividade.

---

# 5. Code - Montar Destinatários

Após consultar os dados no MySQL, um nó `Code` determina quem deverá receber a notificação.

A regra utilizada é:

```text
Aluno >= 18 anos
        │
        └──► Aluno

Aluno < 18 anos
        │
        ├──► Aluno
        │
        └──► Responsável
```

Exemplo de código:

```javascript
const destinatarios = [];

const aluno = {
  id: $json.aluno_id,
  nome: $json.aluno,
  email: $json.email_aluno,
  telefone: $json.telefone_aluno
};

destinatarios.push({
  tipo: 'aluno',
  ...aluno
});

const hoje = new Date();
const nascimento = new Date($json.data_nascimento);

let idade = hoje.getFullYear() - nascimento.getFullYear();

const aniversarioAindaNaoOcorreu =
  hoje.getMonth() < nascimento.getMonth() ||
  (
    hoje.getMonth() === nascimento.getMonth() &&
    hoje.getDate() < nascimento.getDate()
  );

if (aniversarioAindaNaoOcorreu) {
  idade--;
}

if (idade < 18 && $json.responsavel_id) {
  destinatarios.push({
    tipo: 'responsavel',
    id: $json.responsavel_id,
    nome: $json.responsavel,
    email: $json.email_responsavel,
    telefone: $json.telefone_responsavel
  });
}

return destinatarios.map(destinatario => ({
  json: {
    ...$json,
    destinatario
  }
}));
```

---

# 6. Split Out

O `Split Out` separa os destinatários para que cada um seja processado individualmente.

Exemplo:

```text
Aluno 1
   ↓
Aluno 2
   ↓
Responsável do Aluno 2
   ↓
Aluno 3
   ↓
...
```

Isso permite que uma falha em um destinatário não interrompa necessariamente o processamento dos demais.

---

# 7. MySQL - Verificar Duplicidade

Antes de realizar o envio, o workflow consulta o MySQL para verificar se aquela notificação já foi enviada.

Exemplo:

```sql
SELECT id
FROM notificacoes_atividades
WHERE atividade_id = ?
  AND aluno_id = ?
  AND destinatario_tipo = ?
  AND canal = ?;
```

Os parâmetros representam:

* ID da atividade;
* ID do aluno;
* Tipo de destinatário;
* Canal de comunicação.

---

# 8. Controle de E-mail

Um nó `IF` verifica se o destinatário possui endereço de e-mail.

### Possui e-mail

O fluxo segue para o envio.

### Não possui e-mail

O canal de e-mail é ignorado e o fluxo pode continuar para o WhatsApp.

---

# 9. Enviar E-mail

O n8n utiliza um nó de envio de e-mail ou integração SMTP.

Exemplo de conteúdo:

```text
Olá, [Nome].

Uma nova atividade foi cadastrada para sua turma.

Disciplina: [Disciplina]
Atividade: [Descrição]
Data: [Data]

Acesse o sistema de gestão escolar para consultar mais informações.

Atenciosamente,
Coordenação Pedagógica
```

---

# 10. Controle de WhatsApp

Depois do e-mail, o workflow verifica se existe telefone válido para o destinatário.

Caso exista, o fluxo realiza uma requisição para a API de WhatsApp configurada.

O envio pode ser realizado utilizando um nó `HTTP Request`.

A implementação da API dependerá do provedor escolhido.

---

# 11. Enviar WhatsApp

Exemplo de mensagem:

```text
Olá, [Nome]! 📚

Foi cadastrada uma nova atividade para sua turma.

📖 Disciplina: [Disciplina]
📝 Atividade: [Descrição]
📅 Data: [Data]

Acesse o sistema de gestão escolar para consultar mais informações.
```

---

# 12. MySQL - Registrar Resultado

Após cada tentativa de envio, o resultado deve ser registrado no MySQL.

Exemplo:

```sql
INSERT INTO notificacoes_atividades
(
    atividade_id,
    aluno_id,
    destinatario_tipo,
    canal,
    status,
    enviado_em,
    mensagem_erro
)
VALUES
(
    ?,
    ?,
    ?,
    ?,
    ?,
    NOW(),
    ?
);
```

O campo `status` pode assumir valores como:

```text
ENVIADO
ERRO
```

---

# 13. Tratamento de erros

O workflow deve considerar erros individuais durante o processamento.

Exemplos:

* Falha no MySQL;
* Falha no SMTP;
* E-mail inválido;
* Telefone inválido;
* API do WhatsApp indisponível;
* Timeout;
* Dados obrigatórios ausentes.

Sempre que possível, o erro deve ser registrado no banco MySQL.

---

# 14. Tabela de controle

Uma tabela como a seguinte pode ser utilizada para controlar os envios:

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

Para reforçar a idempotência:

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

Essa restrição impede que a mesma combinação de atividade, aluno, destinatário e canal seja registrada mais de uma vez.

---

# 15. Idempotência

A idempotência é importante porque o Schedule Trigger executará o workflow várias vezes.

Sem um mecanismo de controle, uma mesma atividade poderia gerar várias notificações.

O workflow utiliza:

```text
atividade
+
aluno
+
destinatário
+
canal
```

como identificação da notificação.

Antes do envio, o MySQL é consultado.

Se já existir um registro:

```text
Não envia novamente
```

Se não existir:

```text
Envia
↓
Registra no MySQL
```

---

# 16. Credenciais

As credenciais não devem ser armazenadas diretamente no workflow.

O n8n deve utilizar seu sistema de credenciais para armazenar:

### MySQL

* Host;
* Porta;
* Banco;
* Usuário;
* Senha.

### E-mail

* Servidor SMTP;
* Porta;
* Usuário;
* Senha.

### WhatsApp

* URL da API;
* Token;
* Identificadores necessários pelo provedor.

Nenhuma dessas informações deve ser publicada no GitHub.

---

# 17. Segurança

O acesso ao MySQL deve utilizar um usuário específico para o n8n, com apenas as permissões necessárias.

Sempre que possível, recomenda-se separar:

```text
Usuário de leitura
        +
Usuário de escrita
```

de acordo com as necessidades da implementação.

Também devem ser utilizados:

* Senhas fortes;
* Conexões protegidas;
* Controle de acesso;
* Credenciais do n8n;
* Dados fictícios em ambientes de demonstração.

---

# 18. LGPD

Como o workflow trabalha com dados de alunos e responsáveis, devem ser aplicadas boas práticas de proteção de dados.

O workflow deve utilizar apenas os dados necessários para:

* Identificação;
* Comunicação;
* Registro do envio.

Dados pessoais não devem ser incluídos nos arquivos versionados no GitHub.

Para demonstrações, recomenda-se utilizar dados fictícios.

---

# 19. Escalabilidade

Em cenários com grande quantidade de alunos, recomenda-se evitar o processamento de todos os destinatários simultaneamente.

O workflow pode utilizar recursos do n8n como:

* `Loop Over Items`;
* Processamento em lotes;
* Limitação de requisições;
* Controle de concorrência;
* Tratamento de erros.

Isso é especialmente importante quando a API do WhatsApp possui limites de requisições.

---

# 20. Testes

Antes da utilização em produção, recomenda-se testar:

### Teste 1

Aluno maior de idade com e-mail e telefone.

Resultado esperado:

```text
Aluno recebe e-mail.
Aluno recebe WhatsApp.
```

### Teste 2

Aluno menor de idade com responsável cadastrado.

Resultado esperado:

```text
Aluno recebe e-mail.
Aluno recebe WhatsApp.
Responsável recebe e-mail.
Responsável recebe WhatsApp.
```

### Teste 3

Aluno sem e-mail.

Resultado esperado:

```text
E-mail não enviado.
WhatsApp processado normalmente.
```

### Teste 4

Aluno sem telefone.

Resultado esperado:

```text
WhatsApp não enviado.
E-mail processado normalmente.
```

### Teste 5

Notificação já registrada no MySQL.

Resultado esperado:

```text
Não enviar novamente.
```

### Teste 6

Falha na API do WhatsApp.

Resultado esperado:

```text
Status = ERRO
Mensagem de erro registrada no MySQL.
Demais destinatários continuam sendo processados.
```

---

# 21. Resultado esperado

Após a implementação, o processo deverá funcionar automaticamente:

```text
Professor cadastra atividade
          ↓
Atividade fica disponível no MySQL
          ↓
n8n identifica a atividade
          ↓
Busca alunos da turma
          ↓
Identifica responsáveis
          ↓
Aplica regra de idade
          ↓
Verifica duplicidade
          ↓
Envia e-mail
          ↓
Envia WhatsApp
          ↓
Registra resultado no MySQL
```

A Coordenação Pedagógica passa a ter um processo automatizado, rastreável e com controle de duplicidade.

---

# 22. Observação sobre o banco MySQL

Os nomes de tabelas e campos utilizados neste documento são **exemplos didáticos**.

Na implementação real, as consultas devem ser adaptadas ao schema existente no banco de dados MySQL do sistema de gestão escolar.

O workflow não deve assumir nomes de tabelas, relacionamentos ou campos que não tenham sido confirmados no banco real.
