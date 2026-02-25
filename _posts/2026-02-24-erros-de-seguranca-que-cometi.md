---
title: Os Erros de Segurança Que Eu Já Cometi (E Vejo Desenvolvedores Cometendo Todos os Dias)
date: 2026-02-24 22:40
categories: [Segurança, engenharia de software, arquitetura de software]
tags: [segurança, secure coding, jwt, autenticação, autorização, criptografia, hash, salt, bcrypt, argon2, arquitetura, backend, boas práticas, vulnerabilidades]
author: thayran
description: "Uma análise prática e reflexiva sobre os erros de segurança mais comuns que desenvolvedores cometem de JWT eterno a senhas mal protegidas e como pensar em segurança como parte da arquitetura do software"
image: "../assets/images/2026-02-24-erros-de-seguranca-que-cometi/desenvolvimento-seguro.png"
---

Eu já subi código em produção achando que estava tudo certo.

Deploy sem erros.
Feature funcionando.
Cliente feliz.

E mesmo assim… o sistema estava vulnerável.

O problema é que a maioria dos erros de segurança não quebram a aplicação.
Eles ficam ali, silenciosos, esperando alguém curioso o suficiente.

Com o tempo, comecei a perceber um padrão:
não eram falhas “de hacker”, eram falhas de engenharia.

Aqui estão alguns dos erros mais comuns que eu já cometi e que ainda vejo acontecerem com frequência.

---

## 1. "Mas eu já validei no frontend"

Essa frase deveria acender um alerta imediato.

Validação no frontend é experiência do usuário.
Validação no backend é segurança.

Uma vez participei de um projeto onde o campo `role` vinha escondido no formulário.
O backend simplesmente confiava.

Bastava interceptar a requisição e trocar:

```json
{
  "role": "user"
}
```

por

```json
{
  "role": "admin"
}
```

Pronto.

Promoção instantânea.

O frontend não protege nada, ele apenas sugere comportamento.

Se a regra é importante, ela precisa existir no backend.

---

## 2. SQL Injection “moderno”

Muita gente acha que SQL Injection é coisa antiga.

"Eu uso ORM."

Ótimo. Mas você já viu alguém fazer isso?

```javascript
db.raw(`SELECT * FROM users WHERE email = '${email}'`)
```

Bastam alguns segundos para alguém curioso explorar.

O problema não é a tecnologia.
É confiar demais.

Query parametrizada não é detalhe. É obrigação.

---

## 3. Stack trace em produção

Já viu erro assim?

```
Error at UserRepository.js line 87
```

Ou pior:

```
Unhandled Exception: duplicate key value violates unique constraint "users_email_key"
Detail: Key (email)=(teste@email.com) already exists.
```

Cada mensagem de erro detalhada é um mapa para quem está tentando explorar o sistema.

Logs detalhados devem existir.
Mas apenas no servidor.

Para o cliente:

> Ocorreu um erro inesperado.

Simples. Seco. Sem pistas.

---

## 4. Senhas mal protegidas

Eu já vi sistema armazenando senha em texto puro.

Já vi hash com MD5.

Já vi “criptografia própria”.

Isso não é engenharia. É aposta.

Hoje não existe desculpa para não usar:

* bcrypt
* Argon2

E aqui entra um ponto que muita gente ignora: **salt**.

### O que é salt?

Salt é um valor aleatório único que é combinado com a senha antes de gerar o hash.

Em vez de fazer apenas:

```
hash = hash("senha123")
```

Você faz algo como:

```
hash = hash("senha123" + salt)
```

O salt é armazenado junto com o hash no banco.

### Por que isso importa?

Sem salt, dois usuários com a mesma senha terão o mesmo hash.

Isso permite:

* Ataques com rainbow tables
* Identificação de usuários que compartilham senha
* Ataques em massa muito mais rápidos

Com salt:

* Cada hash é único
* Rainbow tables se tornam inúteis
* O atacante precisa quebrar senha por senha

E aqui está o detalhe importante:

Você não precisa implementar salt manualmente se estiver usando bcrypt ou Argon2 corretamente.
Esses algoritmos já geram e armazenam o salt automaticamente.

Ou seja, se você está concatenando salt manualmente antes de usar bcrypt, provavelmente está fazendo errado.

### Outro ponto crítico

Senha nunca deve ser reversível.

Se você consegue descriptografar a senha do usuário, significa que alguém que invadir seu banco também consegue.

Senha deve ser:

* Hasheada
* Com salt
* Com algoritmo resistente a brute force
* Com custo computacional configurado corretamente

E você nunca deveria implementar sua própria solução.

Criptografia caseira é vulnerabilidade com prazo de validade.

Se existe uma área onde você deve confiar em especialistas e padrões consolidados, é essa.

---

## 5. Autenticação não é autorização

Esse é sutil.

Código comum:

```javascript
if (user) {
   return order
}
```

Mas… essa order é dele?

Já vi API onde bastava trocar o ID na URL para acessar dados de outro usuário.

Isso não é falha de login.
É falha de modelagem.

Toda requisição deveria responder três perguntas:

1. Quem é o usuário?
2. O que ele está tentando acessar?
3. Ele pode acessar esse recurso específico?

---

## 6. JWT eterno

Token sem expiração é praticamente um passe vitalício.

Já vi token válido por 365 dias.

Se vazar, acabou.

Segurança é assumir que vazamentos vão acontecer.
E limitar o impacto quando acontecerem.

---

Boa. Concordo com você de novo.

Hoje em dia quase ninguém deixa `/admin` escancarado desse jeito. O problema real é mais sutil — e por isso mais perigoso.

Aqui vai uma versão mais madura e realista dessa seção:

---

## 7. Documentação e endpoints de observabilidade expostos

Esse é mais comum do que parece.

Não é sobre deixar `/admin` aberto.

É sobre coisas como:

* `/swagger` ou `/api-docs` acessíveis em produção
* `/actuator` expondo métricas
* `/metrics` retornando dados internos
* `/health` detalhando dependências
* Painel de filas ou cache sem autenticação forte

Em muitos projetos, isso é habilitado para facilitar o desenvolvimento…
e simplesmente nunca é desativado.

Já vi endpoint de health retornando algo assim:

```json
{
  "status": "UP",
  "database": "UP",
  "redis": "UP",
  "version": "2.3.1",
  "environment": "production"
}
```

Parece inocente.

Mas agora quem está analisando seu sistema já sabe:

* Que você usa Redis
* Que existe um banco específico
* Qual versão da aplicação está rodando
* Que o ambiente é produção

Isso reduz incerteza para quem está mapeando superfícies de ataque.

Swagger aberto também parece inofensivo.

“É só documentação.”

Mas documentação de API é, essencialmente:

* Lista completa de endpoints
* Parâmetros esperados
* Modelos de payload
* Possíveis respostas
* Fluxos de autenticação

É um mapa técnico completo.

E mapa é a primeira coisa que alguém procura.

---

### A abordagem mais madura

Não é sobre esconder tudo.

É sobre controlar acesso.

Boas práticas incluem:

* Proteger documentação com autenticação
* Limitar acesso com VPN quando possível
* Separar ambientes corretamente
* Reduzir nível de detalhe em endpoints públicos
* Não expor versões e tecnologias desnecessariamente

Transparência para o time.
Superfície mínima para o mundo.

---

## 8. Sem rate limit

Sem limitação de requisições você permite:

* Bruteforce de senha
* Enumeração de usuários
* Testes automatizados em massa

E o pior: o sistema continua funcionando normalmente.

---

## O verdadeiro problema

O maior erro não é técnico.

É mental.

Muitos desenvolvedores enxergam segurança como:

> Depois a gente vê isso.

Mas arquitetura não é apenas organização de código.
É modelagem de risco.

Quando você desenha um sistema, você está desenhando também:

* Superfícies de ataque
* Pontos de falha
* Possíveis abusos

Segurança não é paranoia.
É maturidade.

---

## A virada de chave

Eu parei de pensar:

> Será que alguém vai tentar explorar isso?

E comecei a pensar:

> Se eu fosse explorar isso, como faria?

Essa pergunta muda tudo.

Porque no fim, engenharia de software não é só fazer funcionar.

É fazer funcionar mesmo quando alguém tenta quebrar.