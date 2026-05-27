# 📓 Anotações do Laboratório — AWS CloudFormation

> Notas pessoais, dúvidas resolvidas e insights coletados durante as aulas práticas.

---

## Aula 1 — Introdução ao CloudFormation

**O que aprendi:**
- CloudFormation faz parte do ecossistema de IaC (Infrastructure as Code) da AWS
- Alternativas populares: Terraform (multi-cloud), Pulumi, AWS CDK
- A grande vantagem do CloudFormation sobre o CDK é a ausência de código de programação — tudo é declarativo em YAML/JSON

**Dúvida respondida:**
> *Por que usar CloudFormation ao invés de criar os recursos manualmente no Console?*

Resposta: Reprodutibilidade. Um template YAML versionado no Git garante que você possa recriar toda a infraestrutura em minutos, em qualquer conta AWS, de forma idêntica.

---

## Aula 2 — Estrutura do Template

**Seções do template (ordem no arquivo não importa, exceto Resources):**

| Seção | Obrigatória | Uso |
|---|---|---|
| `AWSTemplateFormatVersion` | Não | Sempre usar `'2010-09-09'` |
| `Description` | Não | Documentação inline |
| `Parameters` | Não | Entradas dinâmicas |
| `Mappings` | Não | Tabelas de valores (ex: AMI por região) |
| `Conditions` | Não | Lógica condicional |
| `Resources` | **SIM** | Recursos AWS a criar |
| `Outputs` | Não | Valores exportados |

**Funções intrínsecas mais usadas:**
```yaml
!Ref RecursoOuParametro          # Referencia um recurso/parâmetro
!Sub 'prefixo-${Parametro}'      # Substituição de string
!GetAtt Recurso.Atributo         # Obtém atributo de um recurso
!FindInMap [Mapa, Chave1, Chave2] # Busca em Mappings
!Select [0, !GetAZs '']          # Seleciona item de lista
Fn::Base64: !Sub |               # Codifica em Base64 (UserData)
  #!/bin/bash
  echo "hello"
```

---

## Aula 3 — Stacks e Ciclo de Vida

**Estados possíveis de uma Stack:**

```
CREATE_IN_PROGRESS → CREATE_COMPLETE
                   → CREATE_FAILED → ROLLBACK_IN_PROGRESS → ROLLBACK_COMPLETE

UPDATE_IN_PROGRESS → UPDATE_COMPLETE
                   → UPDATE_ROLLBACK_IN_PROGRESS → UPDATE_ROLLBACK_COMPLETE

DELETE_IN_PROGRESS → DELETE_COMPLETE
                   → DELETE_FAILED
```

**Lição importante:** `ROLLBACK_COMPLETE` não é um erro permanente — a stack existe mas está no estado inicial. Você pode deletá-la e recriar corrigindo o problema.

---

## Aula 4 — Parâmetros e Outputs

**Tipos de parâmetro disponíveis:**
- `String` — Texto livre
- `Number` — Número inteiro ou decimal
- `List<Number>` — Lista de números
- `CommaDelimitedList` — Lista separada por vírgula
- `AWS::EC2::KeyPair::KeyName` — Validado automaticamente pela AWS
- `AWS::EC2::Subnet::Id` — Valida se o subnet existe
- Outros tipos AWS específicos

**Boas práticas com parâmetros:**
```yaml
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]  # Limita opções válidas
    Default: dev                          # Valor padrão
    Description: Ambiente de deploy       # Aparece no Console
    ConstraintDescription: "Deve ser dev, staging ou prod"
```

---

## Aula 5 — Segurança e IAM

**Regra de ouro:** Sempre seguir o princípio do menor privilégio.

- Security Groups são **stateful** (retorno permitido automaticamente)
- NACLs são **stateless** (precisam de regra de entrada E saída)
- Para recursos que criam roles IAM, adicionar `--capabilities CAPABILITY_IAM` ao CLI

---

## Perguntas e Respostas

**P: Posso usar o mesmo template para dev e prod?**
R: Sim! Use `Parameters` para diferenciar e `Conditions` para criar recursos apenas em determinados ambientes.

**P: Como atualizar um recurso sem recriar a instância EC2?**
R: Depende do tipo de mudança. Algumas propriedades permitem atualização "in-place", outras exigem substituição. A documentação de cada recurso indica qual o comportamento (`Update requires: No interruption` / `Replacement`).

**P: O que acontece com os dados no S3 se eu deletar a stack?**
R: Por padrão, a AWS não consegue deletar um bucket S3 não-vazio. Use `DeletionPolicy: Retain` para manter o bucket, ou esvazie-o antes de deletar a stack.

---

## Comandos CLI mais usados

```bash
# Validar template (detecta erros de sintaxe)
aws cloudformation validate-template --template-body file://template.yaml

# Criar stack
aws cloudformation create-stack \
  --stack-name minha-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=dev

# Aguardar conclusão
aws cloudformation wait stack-create-complete --stack-name minha-stack

# Ver eventos (útil para debug)
aws cloudformation describe-stack-events --stack-name minha-stack

# Listar outputs
aws cloudformation describe-stacks \
  --stack-name minha-stack \
  --query 'Stacks[0].Outputs'

# Deletar stack
aws cloudformation delete-stack --stack-name minha-stack
```
