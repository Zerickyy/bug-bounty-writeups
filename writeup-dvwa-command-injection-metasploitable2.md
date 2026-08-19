# Writeup: Command Injection - DVWA - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: DVWA (Damn Vulnerable Web App) rodando em Metasploitable2
- URL: http://192.168.56.10/dvwa/vulnerabilities/exec/
- Nível de segurança: Low

## 1. Identificação da vulnerabilidade
A funcionalidade "Ping for FREE" recebe um endereço IP do
usuário e o repassa diretamente para um comando de shell no
servidor, sem sanitização. Uso do caractere `;` (encadeamento de
comandos no Linux) permite injetar comandos adicionais.

## 2. Prova de conceito
Payload:

    127.0.0.1; whoami

Resultado: além da saída normal do ping, foi retornado o usuário
que executa a aplicação: www-data (usuário padrão do Apache).

## 3. Extração de informações do sistema
Payload:

    127.0.0.1; cat /etc/passwd

Resultado: lista completa de usuários do sistema operacional
extraída via aplicação web, incluindo os mesmos usuários de
serviço já identificados nos writeups de exploração direta
(msfadmin, postgres, mysql, distccd, tomcat55, etc.).

## Impacto
Execução arbitrária de comando no servidor com privilégios do
usuário www-data. Combinado com técnicas de escalação de
privilégio (como vimos no NFS), esse acesso inicial via
aplicação web poderia ser encadeado até obtenção de root
completo no sistema.

## Conclusão
Command Injection presente por concatenação direta de entrada do
usuário em comando de shell. Como correção: nunca construir
comandos de shell a partir de entrada do usuário; usar bibliotecas
de rede nativas da linguagem (em vez de chamar `ping` do sistema),
ou, se inevitável, usar listas de permissão (allowlist) rígidas de
caracteres/valores aceitos e funções de escape apropriadas.
