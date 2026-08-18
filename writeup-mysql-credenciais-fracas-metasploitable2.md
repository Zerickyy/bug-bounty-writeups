# Writeup: MySQL - Credencial Padrão e Extração de Dados - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Metasploitable2 (VM VirtualBox)
- Rede: Internal Network isolada (192.168.56.0/24)

## 1. Reconhecimento
Identificado na varredura inicial com Nmap: MySQL 5.0.51a rodando
na porta 3306/tcp.

## 2. Identificação da falha
Servidor MySQL configurado com usuário root sem senha (credencial
vazia/padrão). Adicionalmente, o servidor não suporta conexão TLS
moderna, exigindo desabilitar SSL no cliente para conectar.

## 3. Acesso e enumeração

    mysql -h 192.168.56.10 -u root --skip-ssl

Conexão bem-sucedida como root sem autenticação. Bancos de dados
disponíveis:

    SHOW DATABASES;
    -> dvwa, metasploit, mysql, owasp10, tikiwiki, tikiwiki195

## 4. Extração de dados sensíveis
Dentro do banco `dvwa`, tabela `users`, extraídas 5 contas de
usuário com senhas em hash MD5 (sem salt):

    USE dvwa;
    SELECT * FROM users;

Observação: os hashes dos usuários "admin" e "smithy" são
idênticos, indicando reuso de senha entre contas distintas.

## 5. Quebra de hash (crack offline)
Hash MD5 testado contra a wordlist rockyou.txt usando John the
Ripper:

    john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

Resultado: senha quebrada instantaneamente = "password"

## Conclusão
Cadeia completa de falhas: (1) credencial de banco padrão/vazia
permitiu acesso root sem autenticação; (2) dados sensíveis de
usuários foram extraídos diretamente; (3) hashes MD5 sem salt e
senhas fracas permitiram quebra offline em segundos. Como
correção: nunca usar senha vazia em contas administrativas de
banco, aplicar hashing forte com salt (bcrypt/argon2) em vez de
MD5 puro, e impor política de senha forte para usuários finais.
